# SiI902x HDMI Bridge Driver Modernization

Reference implementation for AM62P5-SK HDMI display and audio using
the modern DRM_BRIDGE_OP_HDMI framework and audio-graph-card2.

Patches are based on **upstream linux-next**. Tested on TI AM62P5-SK.

---

## Overview

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
    0001-drm-bridge-sii902x-Extract-helpers-for-power-state-a.patch
    0002-drm-bridge-sii902x-Convert-to-DRM_BRIDGE_OP_HDMI-fra.patch
    0003-drm-bridge-sii902x-Add-D3-Cold-power-management-via-.patch
    0004-drm-bridge-sii902x-Fix-HPD-notify-tmds_char_rate_val.patch
  audio/
    0005-ASoC-ti-davinci-mcasp-Add-audio-graph-card2-DPCM-sup.patch
overlays/
  k3-am62p5-sk-hdmi-audio.dtso
```

---

## Setting up the branch

Clone linux-next and create a working branch:

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
cd linux-next
git checkout -b sii902x-hdmi-refactor
```

---

## Applying the patches

### Driver patches

```bash
git am patches/driver/0001-*.patch
git am patches/driver/0002-*.patch
git am patches/driver/0003-*.patch
git am patches/driver/0004-*.patch
```

### McASP audio-graph-card2 patch

Required only if using the HDMI audio overlay with `audio-graph-card2`:

```bash
git am patches/audio/0005-*.patch
```

### HDMI audio overlay

Copy the overlay into the kernel DTS tree and register it in the build:

```bash
cp overlays/k3-am62p5-sk-hdmi-audio.dtso \
   arch/arm64/boot/dts/ti/

# Add to arch/arm64/boot/dts/ti/Makefile:
#   dtb-$(CONFIG_ARCH_K3) += k3-am62p5-sk-hdmi-audio.dtbo
```

---

## Build

```bash
export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64

make defconfig ti_arm64_prune.config

scripts/config --enable CONFIG_DRM
scripts/config --enable CONFIG_DRM_TIDSS
scripts/config --enable CONFIG_DRM_SII902X
scripts/config --enable CONFIG_DRM_DISPLAY_CONNECTOR

make Image dtbs modules -j$(nproc)
```

---

## Deploy

```bash
~/scp_linux_am62p.sh <BOARD_IP>

# Copy overlay to boot partition on the board:
scp arch/arm64/boot/dts/ti/k3-am62p5-sk-hdmi-audio.dtbo \
    root@<BOARD_IP>:/boot/
```

Enable the overlay in `/boot/uEnv.txt`:

```
name_overlays=k3-am62p5-sk-hdmi-audio.dtbo
```

Remove any TLV320 audio overlay from `name_overlays` when using HDMI
audio — the HDMI overlay disables the `codec_audio` sound card.

---

## Test

```bash
# Display
dmesg | grep -i "tidss\|sii902\|drm"
cat /sys/class/drm/card*-HDMI-A-1/status

# Audio
aplay -l
cat /proc/asound/card*/eld*
aplay /path/to/audio.wav

# Infoframes (requires debugfs)
mount -t debugfs none /sys/kernel/debug 2>/dev/null
cat /sys/kernel/debug/dri/0/HDMI-A-1/infoframes

# Suspend/resume (D3 Cold)
rtcwake -m mem -s 5
```

---

## Architecture notes

### DRM_BRIDGE_OP_HDMI

Sets `bridge.type = DRM_MODE_CONNECTOR_HDMIA` so
`drm_bridge_connector_init()` creates the HDMI connector automatically.
AVI infoframes are generated and replayed by the framework on every
modeset. The audio codec device is created by
`drm_connector_hdmi_audio_init()` parented to the bridge device, with
its `of_node` pointing to the sii9022 DT node — enabling OF-graph
discovery by audio-graph-card2.

### D3 Cold via atomic callbacks

The D3 Cold entry sequence (datasheet §6.9.1.1) requires a hardware
reset followed by TPI re-initialisation. `dev_pm_ops` cannot be used
because the I2C parent suspends before the DRM bridge in the PM
ordering. DRM atomic callbacks (`atomic_disable`/`atomic_enable`) run
inside the display pipeline before any bus parent suspends, keeping
register access valid throughout.

### audio-graph-card2

`audio-graph-card2` uses OF-graph endpoint traversal to match
endpoints to DAIs, which works with the dynamically-created
`hdmi-audio-codec` platform device that shares the sii9022 `of_node`.
The McASP driver's `MCASP_GRAPH_PORTS` mode registers one DAI for a
single `ports { port@0 }` structure. The sii9022 audio port at
`port@3` (reg=3) is matched by `get_dai_id`
(`SII902X_AUDIO_PORT_INDEX = 3`).

---

## References

- SiI9022A Programming Guide (Silicon Image AN-1021)
- `include/drm/drm_bridge.h` — `DRM_BRIDGE_OP_HDMI` ops
- `drivers/gpu/drm/display/drm_hdmi_audio_helper.c`
- `sound/soc/generic/audio-graph-card2.c`
