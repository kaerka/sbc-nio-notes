# sbc-nio-notes

Lab notes from bringing up the [Radxa NIO 12L](https://docs.radxa.com/en/nio/12l)
(MediaTek MT8395 / Genio 1200). Written so another NIO owner does not have to
rediscover the same traps.

This is a **public notes repo**. It does not include homelab addresses, Wi‑Fi
SSIDs, credentials, private keys, or flash-host layout.

## Goals

- Portable / headless Arm64 SBC
- Edge AI / NPU (APU) feasibility
- Camera / USB audio / display bring-up

## Non-goals

- Chasing vendor-only blobs indefinitely
- Month-long kernel archaeology for its own sake
- Treating this as a Raspberry Pi replacement

## Board (d16 / 16 GB SKU)

| | |
|---|---|
| SoC | MediaTek **MT8395** (Genio 1200), U-Boot `board=mt8195` |
| CPU | 4× Cortex-A78 + 4× Cortex-A55 (AArch64) |
| RAM | 16 GB LPDDR5 — U-Boot overlay **`ddr-16g.dtbo` is required** |
| Storage | 256 GB UFS 3.1 (Samsung `KLUEG8U1EA-B0C1`), **no eMMC**, no NVMe option |
| Ethernet | 1 GbE (`dwmac-mediatek` + RTL8211F PHY on Armbian) |
| Wi‑Fi | onboard MediaTek **MT7921** PCIe (`mt7921e`), 2.4 + 5 GHz |
| Video out | HDMI; USB‑C DP Alt Mode exists in hardware, see [OS notes](docs/os-notes.md) |
| HDMI in | present on the board; early testing looked like pass-through, **not validated** |
| Camera | dual 4-lane MIPI CSI (second connector on the underside) |
| USB | USB‑C OTG + Type‑A via **VIA VL805** PCIe xHCI |
| Serial | **921600 8N1**, not 115200 |

The storage limit (UFS only, no M.2 NVMe) is still the main hardware drawback.

## Start here

1. [Hardware quirks](docs/hardware-quirks.md) — serial, USB host, Wi‑Fi scan, APU, audio, CSI
2. [Boot, UFS, FIT](docs/boot-and-storage.md) — named partitions, download mode, fitImage
3. [OS notes](docs/os-notes.md) — Armbian vs Ubuntu/Rity vs custom Yocto
4. [Related work](docs/related-work.md) — lessons from [zlorenzini/radxa-nio-12l](https://github.com/zlorenzini/radxa-nio-12l) (attributed, not copied)

Raw early CSI/V4L dump: [nio-v4l-csi-ubuntu-mtk.txt](nio-v4l-csi-ubuntu-mtk.txt)

## Short version

- Console is **921600**. 115200 is garbage.
- Do **not** flash a single GPT “disk image” like a Pi. UFS partitions are
  named and written separately (`kernel` is a FIT, `rootfs` is ext4, BL2 lives
  in the UFS boot area).
- Use the **Radxa NIO 12L d16** Rity/Yocto image, not the generic Genio EVK
  image (wrong DTB).
- Type‑A USB host needs DTS work (`dr_mode = "host"`, drop `usb-role-switch`,
  enable `ssusb1`/`xhci2`) **and** `CONFIG_USB_XHCI_PCI=y` for the VL805.
- **APU/NPU is not present on Armbian** (mainline-ish 6.19). `/dev/apusys` needs
  MediaTek’s AIoT BSP kernel, not the public Radxa 5.15 tree either.
- Armbian names Ethernet **`end0`**. At 1 Gbps we hit a **`FSMPES` MAC reset
  loop**. HDMI DRM is **`HDMI-A-1` only**; USB‑C DP Alt Mode is DT/TCPC work,
  not a package. 5 GHz Wi‑Fi scan works once you use **root `iw`**.
- Yocto MediaTek packers want an **x86_64** host. BitBake produces a raw
  `Image`; you wrap a FIT with `mkimage` (gzip kernel, load `0x64000000`).

Deeper recipes live in [project-argus](https://github.com/kaerka/project-argus)
(`meta-radxa-nio12l`). These notes are the condensed “what we actually hit.”

## References

- Radxa: <https://docs.radxa.com/en/nio/12l>
- Rity / Yocto images: <https://dl.radxa.com/nio12l/images/yocto/>
- Public kernel mirror: <https://github.com/radxa/kernel> branch `mtk-v5.15-dev-rity-kirkstone-v24.1`
- MediaTek AIoT (APU): <https://gitlab.com/mediatek/aiot/bsp/linux> (`mtk-v5.15-dev`)
