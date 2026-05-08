# SiI902x HDMI Bridge Driver Modernization

Reference implementation for AM62P5-SK HDMI display and audio using
the modern DRM_BRIDGE_OP_HDMI framework and audio-graph-card2.

Patches are based on **ti-linux-6.18.y**. Tested on TI AM62P5-SK.

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
  audio/
    0005-ASoC-ti-davinci-mcasp-Add-audio-graph-card2-DPCM-sup.patch
overlays/
  k3-am62p5-sk-hdmi-audio.dtso
```

---

## Setting up the branch

```bash
git clone git://git.ti.com/ti-linux-kernel/ti-linux-kernel.git
cd ti-linux-kernel
git checkout ti-linux-6.18.y
git checkout -b sii902x-hdmi-refactor
```

---

## Applying the patches

### Driver patches

```bash
git am patches/driver/0001-*.patch
git am patches/driver/0002-*.patch
git am patches/driver/0003-*.patch
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

## References

- SiI9022A Programming Guide (Silicon Image AN-1021)
- `include/drm/drm_bridge.h` — `DRM_BRIDGE_OP_HDMI` ops
- `drivers/gpu/drm/display/drm_hdmi_audio_helper.c`
- `sound/soc/generic/audio-graph-card2.c`
