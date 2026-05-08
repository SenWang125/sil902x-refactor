# SiI902x HDMI Bridge Driver Modernization

Patches for AM62P5-SK HDMI display and audio. Based on **ti-linux-6.18.y**.

| Area | Before | After |
|---|---|---|
| Display | Manual AVI infoframe writes | `DRM_BRIDGE_OP_HDMI` |
| Audio | `hdmi-codec` pdev at probe | `DRM_BRIDGE_OP_HDMI_AUDIO` |
| Power | `dev_pm_ops` | D3 Cold via DRM atomic callbacks |
| Sound card | `simple-audio-card` | `audio-graph-card2` |

---

## Setup

```bash
git clone git://git.ti.com/ti-linux-kernel/ti-linux-kernel.git
cd ti-linux-kernel
git checkout ti-linux-6.18.y
git checkout -b sii902x-hdmi-refactor
```

## Apply

```bash
git am patches/driver/0001-*.patch
git am patches/driver/0002-*.patch
git am patches/driver/0003-*.patch
git am patches/audio/0005-*.patch   # required for audio-graph-card2

cp overlays/k3-am62p5-sk-hdmi-audio.dtso arch/arm64/boot/dts/ti/
# Add to arch/arm64/boot/dts/ti/Makefile:
#   dtb-$(CONFIG_ARCH_K3) += k3-am62p5-sk-hdmi-audio.dtbo
```

## Build

```bash
export CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm64
make defconfig ti_arm64_prune.config
scripts/config --enable CONFIG_DRM
scripts/config --enable CONFIG_DRM_TIDSS
scripts/config --enable CONFIG_DRM_SII902X
scripts/config --enable CONFIG_DRM_DISPLAY_CONNECTOR
make Image dtbs modules -j$(nproc)
```

## Deploy

```bash
~/scp_linux_am62p.sh <BOARD_IP>
scp arch/arm64/boot/dts/ti/k3-am62p5-sk-hdmi-audio.dtbo root@<BOARD_IP>:/boot/
```

In `/boot/uEnv.txt`:
```
name_overlays=k3-am62p5-sk-hdmi-audio.dtbo
```

## Test

```bash
dmesg | grep -i "tidss\|sii902\|drm"
aplay -l && aplay /path/to/audio.wav
rtcwake -m mem -s 5   # suspend/resume D3 Cold
```
