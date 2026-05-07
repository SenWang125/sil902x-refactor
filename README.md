# SiI902x HDMI Bridge Driver Modernization

Reference implementation for AM62P5-SK HDMI display and audio using
the modern DRM_BRIDGE_OP_HDMI framework and audio-graph-card2.

Tested on TI AM62P5-SK with kernel **ti-linux-6.18.y**.

---

## Overview

The SiI9022A HDMI bridge driver in the TI vendor tree uses legacy
patterns: manual AVI infoframe packing, an `hdmi-codec` platform device
created at probe time, and system/runtime PM hooks. This repository
contains patches that replace those patterns with the modern DRM and
ASoC framework equivalents.

| Area | Before | After |
|---|---|---|
| Display | Manual AVI infoframe writes | `DRM_BRIDGE_OP_HDMI` — framework managed |
| Audio | `hdmi-codec` pdev at probe | `DRM_BRIDGE_OP_HDMI_AUDIO` — connector-attached codec |
| Power | `dev_pm_ops` suspend/resume | D3 Cold via DRM `atomic_disable`/`atomic_enable` |
| Sound card | `simple-audio-card` | `audio-graph-card2` with OF-graph |

---

## Contents

```
patches/
  driver/
    0000-cover-letter.patch
    0001-drm-bridge-sii902x-Extract-helpers.patch
    0002-drm-bridge-sii902x-Convert-to-DRM_BRIDGE_OP_HDMI.patch
    0003-drm-bridge-sii902x-D3-Cold-PM.patch
  audio/
    0005-ASoC-ti-davinci-mcasp-Add-audio-graph-card2-DPCM-sup.patch
overlays/
  k3-am62p5-sk-hdmi-audio.dtso
```

---

## Prerequisites

- TI vendor kernel: `ti-linux-6.18.y` (tested on 6.18.13)
- Board: AM62P5-SK (`k3-am62p5-sk.dts`)
- Cross-compiler: `aarch64-linux-gnu-`
- `AM62P5-SK` display DTS already populated (DSS0, McASP1 pins)

---

## Patches

### Driver patches (`patches/driver/`)

Apply in order to `drivers/gpu/drm/bridge/sii902x.c`.

**Patch 1 — Extract helpers**

Extracts `sii902x_set_power_state()`, `sii902x_clear_interrupts()`, and
`sii902x_reset()` as standalone helpers. No functional change; prepares
for Patches 2 and 3.

**Patch 2 — DRM_BRIDGE_OP_HDMI conversion**

The core change. Replaces the legacy approach with the modern bridge
framework:

- Sets `DRM_BRIDGE_OP_HDMI | DRM_BRIDGE_OP_HDMI_AUDIO` on the bridge.
- Implements the four mandatory infoframe callbacks:
  `hdmi_write_avi_infoframe`, `hdmi_clear_avi_infoframe`,
  `hdmi_write_hdmi_infoframe` (no-op — SiI9022A TPI has no dedicated
  VSIF registers), `hdmi_clear_hdmi_infoframe`.
- Adds `hdmi_write_audio_infoframe` / `hdmi_clear_audio_infoframe` for
  HDMI audio infoframes.
- Implements audio callbacks: `hdmi_audio_startup`, `hdmi_audio_prepare`,
  `hdmi_audio_shutdown`, `hdmi_audio_mute_stream`.
- Sets `bridge.vendor = "SiI"` and `bridge.product = "SiI9022"`.
  **Note:** `vendor` must not exceed 8 characters
  (`DRM_CONNECTOR_HDMI_VENDOR_LEN`). "Silicon Image" (13 chars) will
  cause `drmm_connector_hdmi_init()` to return `-EINVAL`.
- Removes the manually created `hdmi-codec` platform device; the
  framework creates it via `drm_connector_hdmi_audio_init()`.
- Replaces `sii902x_bridge_mode_set` + `sii902x_bridge_enable` with
  a single `atomic_enable`. Output sequence is:
  1. Exit D3 if suspended (reset, restore TPI and interrupt config)
  2. Set output mode (HDMI/DVI) and enter D0
  3. Write video timing registers
  4. Write infoframes (via `drm_atomic_helper_connector_hdmi_update_infoframes`)
  5. Clear `SYS_CTRL_PWR_DWN` to start TMDS

**Patch 3 — D3 Cold power management**

Adds D3 Cold entry/exit via DRM atomic callbacks instead of PM ops:

- `atomic_disable`: saves TPI and interrupt context, issues hardware
  reset, enables HPD-only interrupt, writes D3 power state.
- `atomic_enable`: detects `d3_suspended` flag, re-initialises TPI,
  restores interrupt mask, then proceeds with normal enable.
- IRQ handler: uses `drm_bridge_hpd_notify()` (modern) instead of
  `drm_helper_hpd_irq_event()` (legacy). In D3, reports
  `connector_status_connected` to wake the system without register
  access.
- No `dev_pm_ops`, no runtime PM — all state transitions go through
  the DRM display pipeline.

### Audio patch (`patches/audio/`)

**0005 — McASP audio-graph-card2 DPCM support**

Adds OF-graph and audio-graph-card2 support to `davinci-mcasp`. Three
modes are auto-detected at probe:

| Mode | DT structure | DAIs |
|---|---|---|
| `MCASP_GRAPH_NONE` | No OF-graph endpoints | 1 (legacy) |
| `MCASP_GRAPH_PORT` | `port { endpoint }` | 1 |
| `MCASP_GRAPH_PORTS` | `ports { port@N }` | 1 (non-DPCM) |
| `MCASP_GRAPH_DPCM` | `ports { port@N }` + `dpcm` container | N |

Adds `of_xlate_dai_id` to the component driver so `audio-graph-card2`
can resolve endpoints to DAI IDs without requiring `#sound-dai-cells`.

---

## Applying the patches

### 1. Apply driver patches

```bash
cd <kernel-tree>

git am patches/driver/0001-drm-bridge-sii902x-Extract-helpers.patch
git am patches/driver/0002-drm-bridge-sii902x-Convert-to-DRM_BRIDGE_OP_HDMI.patch
git am patches/driver/0003-drm-bridge-sii902x-D3-Cold-PM.patch
```

If `git am` fails due to context differences between vendor and upstream,
apply manually:

```bash
patch -p1 --dry-run < patches/driver/0002-...patch   # check first
patch -p1            < patches/driver/0002-...patch
```

### 2. Apply McASP audio patch

```bash
git am patches/audio/0005-ASoC-ti-davinci-mcasp-Add-audio-graph-card2-DPCM-sup.patch
```

### 3. Add HDMI audio overlay to the kernel build

Copy the overlay to the DTS directory:

```bash
cp overlays/k3-am62p5-sk-hdmi-audio.dtso \
   arch/arm64/boot/dts/ti/
```

Add it to the Makefile so it gets built:

```bash
# In arch/arm64/boot/dts/ti/Makefile, find the AM62P5 overlay block and add:
dtb-$(CONFIG_ARCH_K3) += k3-am62p5-sk-hdmi-audio.dtbo
```

---

## DTS changes required for display

The patches above cover only the driver and audio overlay. For the
**HDMI display pipeline** on AM62P5-SK, the following DTS nodes must be
present in the vendor kernel tree. Most are already in `ti-linux-6.18.y`
but are listed here for reference.

### `k3-am62p-j722s-common-main.dtsi`

- `dss_clk_ctrl` syscon node
- `dss_oldi_io_ctrl` syscon node
- `timesync_router` node
- `dss0` / `dss1` DSS platform nodes with OLDI transmitter ports
- `dss0_vp1_clk` / `dss1_vp1_clk` clock nodes

### `k3-am62p5-sk.dts`

- `framebuffer0` in `/chosen` (for early boot display)
- `linux,cma` memory pool
- `main_dpi_pins_default` pin configuration (29 VOUT0 pins)
- `sii9022` bridge node in `&main_i2c1`:
  ```dts
  sii9022: bridge-hdmi@3b {
      compatible = "sil,sii9022";
      reg = <0x3b>;
      interrupt-parent = <&exp1>;
      interrupts = <16 IRQ_TYPE_EDGE_FALLING>;
      reset-gpios = <&exp1 ... GPIO_ACTIVE_LOW>;
      #sound-dai-cells = <0>;
      sil,i2s-data-lanes = <0>;

      hdmi_tx_ports: ports {
          port@0 { sii9022_in: endpoint { remote-endpoint = <&dss0_dpi1_out>; }; };
          port@1 { sii9022_out: endpoint { remote-endpoint = <&hdmi_connector_in>; }; };
      };
  };
  ```
- `connector-hdmi` node (for legacy display-connector bridge)
- `&dss0` enabled with `dss0_dpi1_out` endpoint pointing to `sii9022_in`

### `tidss_drv.c`

Upstream `tidss` does not include `ti,am62p-dss` in its match table as
of the current linux-next. One line is needed:

```c
/* In drivers/gpu/drm/tidss/tidss_drv.c, device_id table: */
{ .compatible = "ti,am62p-dss", .data = &dispc_am625_feats },
```

---

## Build

```bash
cd <kernel-tree>
source ~/.venv/linux/bin/activate        # Python venv for DT tools if needed

export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64

make defconfig ti_arm64_prune.config

# Ensure these are built-in (not modules):
scripts/config --enable CONFIG_DRM
scripts/config --enable CONFIG_DRM_TIDSS
scripts/config --enable CONFIG_DRM_SII902X
scripts/config --enable CONFIG_DRM_DISPLAY_CONNECTOR

make Image -j$(nproc)
make dtbs
make modules -j$(nproc)
```

---

## Deploy

```bash
# SCP kernel, modules, and DTB to the board (replace <IP>):
~/scp_linux_am62p.sh <IP>

# Copy the HDMI audio overlay to the board's boot partition:
scp arch/arm64/boot/dts/ti/k3-am62p5-sk-hdmi-audio.dtbo \
    root@<IP>:/boot/
```

Enable the overlay in `uEnv.txt` on the board:

```
# /boot/uEnv.txt — add to name_overlays:
name_overlays=k3-am62p5-sk-hdmi-audio.dtbo
```

If using the TLV320 audio codec overlay simultaneously, that overlay
must be removed from `uEnv.txt` — the HDMI audio overlay disables the
`codec_audio` sound card.

---

## Test

### Display

```bash
# Confirm DSS and sii902x probed:
dmesg | grep -i "tidss\|sii902\|drm"

# DRM connector status:
cat /sys/class/drm/card*-HDMI-A-1/status   # should print "connected"
```

### HDMI audio

```bash
# Confirm ALSA card registered:
aplay -l

# Check ELD (should show monitor name and audio formats, not all zeros):
cat /proc/asound/card*/eld*

# Play audio:
aplay /path/to/audio.wav
```

### Infoframes (requires debugfs)

```bash
mount -t debugfs none /sys/kernel/debug 2>/dev/null
cat /sys/kernel/debug/dri/0/HDMI-A-1/infoframes
```

### Suspend/resume (D3 Cold)

```bash
# Suspend for 5 seconds — display should turn off and resume cleanly:
rtcwake -m mem -s 5

# Confirm D3 Cold was entered and exited:
dmesg | grep -i "sii902\|d3\|suspend\|resume"
```

---

## Architecture notes

### Why DRM_BRIDGE_OP_HDMI

The `DRM_BRIDGE_OP_HDMI` framework (introduced around Linux 6.7)
centralises infoframe management and HDMI connector creation. Benefits:

- AVI infoframes are generated and replayed by the framework on every
  modeset; the driver only writes to hardware registers.
- `drm_bridge_connector_init()` creates the HDMI connector automatically
  when the display controller uses `DRM_BRIDGE_ATTACH_NO_CONNECTOR`
  (as tidss does). No separate `hdmi-connector` DT node needed if the
  bridge declares `bridge.type = DRM_MODE_CONNECTOR_HDMIA`.
- Audio codec device is created by `drm_connector_hdmi_audio_init()`
  after connector creation, parented to the bridge device. The codec's
  `of_node` inherits the bridge device's `of_node`, enabling OF-graph
  audio card discovery.

### Why D3 Cold via atomic callbacks

The SiI9022A D3 Cold entry sequence (datasheet §6.9.1.1) requires a
hardware reset followed by TPI re-initialisation. This cannot be done
in `dev_pm_ops` because:

1. I2C parent suspends before the DRM bridge in the PM ordering.
2. Once I2C is suspended, register access fails.
3. The DRM atomic callbacks (`atomic_disable`/`atomic_enable`) run
   inside the display pipeline teardown/bring-up, before any bus
   parent suspends.

### Why audio-graph-card2

`simple-audio-card` identifies CPU and codec DAIs via `sound-dai`
phandles. With `DRM_BRIDGE_OP_HDMI_AUDIO`, the audio codec is a
dynamically-created platform device whose `of_node` is the bridge's
`of_node`. `audio-graph-card2` uses OF-graph endpoint traversal
(`of_xlate_dai_id`) to match endpoints to DAIs, which works correctly
with dynamically created devices sharing an `of_node`.

The McASP driver's `MCASP_GRAPH_PORTS` mode registers one DAI for the
single `ports { port@0 }` structure. The sii9022 audio port at
`port@3` (reg=3) is matched by `get_dai_id` in the DRM HDMI audio
helper (`SII902X_AUDIO_PORT_INDEX = 3`).

---

## References

- SiI9022A Programming Guide (Silicon Image AN-1021)
- `Documentation/gpu/drm-kms-helpers.rst` — bridge framework overview
- `include/drm/drm_bridge.h` — `DRM_BRIDGE_OP_HDMI` ops
- `drivers/gpu/drm/display/drm_hdmi_audio_helper.c` — audio codec init
- `sound/soc/generic/audio-graph-card2.c` — card topology documentation
