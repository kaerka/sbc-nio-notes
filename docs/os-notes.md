# OS notes

Three stacks got real hours on this board.

## Ubuntu / Rity (vendor)

MediaTek **Rity Yocto** (Kirkstone, kernel 5.15.47) is the known-good
**boot chain**: BL2, FIP, FIT, UFS labels, DTBOs.

- Image: NIO **d16** tarball from
  <https://dl.radxa.com/nio12l/images/yocto/> — not the generic EVK.
- Good for: serial, UFS layout, `boot_conf`, extracting `fitImage` / DTBOs /
  `boot.script` as a template.
- `uname -r` on the reference kernel we booted:
  `5.15.47-mtk+g257d19099297`.

Build **host** for MediaTek’s IoT Yocto SDK: **x86_64** (Ubuntu 22.04 is what
the packers expect). aarch64 hosts fail on prebuilt x86 firmware/signing
tools. Cross-compiling the aarch64 kernel itself is fine; the **host
binaries** are the trap.

## Armbian (edge-genio)

Second NIO unit, Aug 2026. Image:

`Armbian_26.2.4_Radxa-nio-12l_trixie_edge_6.19.8-ufs_minimal`

then `6.19.8-edge-genio` packages. `BOARD=radxa-nio-12l`,
`BOOT_SOC=mt8395`. Flash with **`mtk-flash`** (BROM → DA → fastboot), not a
raw `dd` of a GPT card image. Default hostname on that image: `radxa-nio-12l`.

| Area | Result |
|---|---|
| Boots to userspace | yes |
| HDMI / Panfrost | `/dev/dri` (`card0`, `card1`, `renderD128`); DRM connector **`HDMI-A-1` only** |
| linuxkms / GBM UI | works on HDMI (no desktop required) |
| Wi‑Fi scan (privileged) | 2.4 + 5 GHz; unprivileged scan is a false negative |
| 5 GHz STA join | not completed on Armbian (Ubuntu on the **same board** did join 5 GHz) |
| APU / `/dev/apusys` | **MISSING** — clocks/reserved-memory only; empty overlay dir |
| USB‑C DP Alt Mode | hardware yes; this image **no** — TCPC fail, `dp-tx` disabled |
| USB gadget ethernet | not observed |
| GbE `end0` @ 1 G | **FSMPES** parity error → MAC reset loop; second cable same |
| `armbian-firmware-full` | unpack **hard-locked** the board (including serial); power cycle |
| Netplan | networkd, not NM; `10-dhcp-all-interfaces.yaml` stomps static `end0` |

Armbian is a reasonable HDMI “SSH + kiosk” box **if** you live on Wi‑Fi. It is
a **bad** vehicle for APU, USB‑C DP, Gigabit Ethernet, or a locked-down UFS
flash workflow.

## Custom Yocto (Kirkstone)

Public kernel: `github.com/radxa/kernel`, branch
`mtk-v5.15-dev-rity-kirkstone-v24.1`, pinned
`SRCREV=b805e4e7f0ac790766ed8dc95443133c7e2f22cb`.

Feed that tree a **full** Rity `/proc/config.gz` as `file://defconfig` (Yocto
`make oldconfig`, not `make defconfig`). Generic `genio_1200_*_defconfig` in
the public tree built **UFS as a module** and hung before mounting root.

Practical kernel promotions we needed beyond Rity’s running config:

| Symbol | Why |
|---|---|
| `USB_XHCI_PCI=y` | VL805 Type‑A |
| `SND_USB_AUDIO=y` | ReSpeaker |
| `MTK_SCP`, `DRM_PANFROST`, `MEDIA_SUPPORT`, `VIDEO_DEV`, `MTK_SVS` as `=y` | avoid module-at-boot gaps |
| `HW_RANDOM_MTK` | asked for `=y`; **`make oldconfig` kept `=m`** — dependency, not a forgotten edit |

BitBake output is `Image` + DTB + `modules-*.tgz`. Wrap a FIT as in
[boot-and-storage](boot-and-storage.md). Flash FIT to `kernel`, modules onto
the matching rootfs.

Recipes: [kaerka/project-argus](https://github.com/kaerka/project-argus)
`yocto/layers/meta-radxa-nio12l`. APU still waits on the MediaTek AIoT kernel
(`mtk-v5.15-dev`), not this public Radxa fork.
