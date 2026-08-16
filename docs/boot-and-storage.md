# Boot, UFS, and FIT images

The NIO 12L does **not** boot like a Raspberry Pi (one GPT `.img` on a card).
It uses **named UFS partitions** written by `genio-flash` from a JSON manifest.
Each payload is a separate blob.

## Reference image

Use the **NIO 12L d16** Rity/Yocto tarball, not the generic Genio EVK image.

Example (Jan 2025):

```
genio-1200-radxa-nio-12l-d16-ufs-kirkstone-k5.15-v24.0-b3-20250109.tar.gz
https://dl.radxa.com/nio12l/images/yocto/
```

The Jan 2024 EVK package (`genio-1200-evk-ufs-2024`) carries
`mediatek_genio-1200-evk-ufs.dtb` and **will not** boot this board correctly.

U-Boot identity we saw on hardware:

```
board=mt8195
storage=scsi
storage_dev=2
dtb_path=/FIRMWARE/mediatek/genio-1200-radxa-nio-12l-d16/
baudrate=921600
```

## Partition map (Rity `rity.json` / `.wks`)

| Label / area | Typical size | Payload |
|---|---|---|
| `mmc0boot0` (UFS boot area, not GPT) | — | BL2 preloader (`bl2.img`) |
| `mmc0boot1` | — | U-Boot environment |
| `bootloaders` / `bootloaders_b` | 4 MB | FIP (BL31 + U-Boot) |
| `firmware` / `firmware_b` | 32 MB | optional HW blobs |
| `dramk` | 512 KB | DRAM calibration |
| `misc` | 1 MB | flags |
| `bootassets` | 32 MB | boot splash VFAT |
| `EFI_system_partition` / capsule | 100 MB | EFI/OTA capsule VFAT |
| **`kernel`** | **32 MB** | **FIT image** (not a raw `Image`) |
| **`rootfs`** | rest | ext4; cmdline `root=PARTLABEL=rootfs rootwait` |

Rootfs GPT type GUID used by Rity: `B921B045-1DF0-41C3-AF44-4C6F280D3FAE`
(ARM64 Linux root).

Linux enumerates UFS as SCSI. Well-known LUNs plus the user device show up as
`sda`/`sdb`/`sdc`; **always prefer `PARTLABEL=`** (`lsblk -o NAME,SIZE,PARTLABEL`).
The `kernel` partition is small — a 32 MB FIT must actually fit.

Iterating the kernel: `dd` the new FIT onto the **`kernel` PARTLABEL** from a
running system, `sync`, reboot. Keep **modules in lockstep** with that FIT
(extract `modules-*.tgz` into `/lib/modules/$(uname -r)` or rebuild the
rootfs). A new `Image` without matching modules will boot and then fail
randomly on out-of-tree / module drivers.

## FIT layout

Rity / custom FITs we build:

| | |
|---|---|
| `bootscr-boot.script` | U-Boot script, uncompressed |
| `kernel-1` | Linux **gzip**-compressed, load/entry **`0x64000000`** |
| `fdt-mediatek_genio-1200-radxa-nio-12l.dtb` | base DTB, load `0x44000000` |
| `fdt-*.dtbo` | overlays, load `0x44c00000` |

Yocto `kernel.bbclass` on this BSP emits a raw ARM64 **`Image`**, not a FIT.
Repack with `mkimage -f foo.its` after `gzip -9` the `Image` to `linux.bin`.
DTBOs are easier copied from the Rity `devicetree/` directory than extracted
from an old FIT.

Kernel we actually compiled from source: Radxa public
`mtk-v5.15-dev-rity-kirkstone-v24.1` at
`b805e4e7f0ac790766ed8dc95443133c7e2f22cb` (Rity’s older
`257d19099297` tip was force-pushed off the remote). **`CONFIG_SCSI_UFS_MEDIATEK`
must be `=y`**, not `=m`, or the rootfs is unreachable (no initramfs).

## Fastboot vs UFS

U-Boot **fastboot talks MMC**. Storage is **UFS/SCSI**. `fastboot usb 0` writes
to the wrong abstraction and fail in practice.

Use:

- **`genio-flash`** in BROM/DA mode for full brick-level flashes, or
- **Linux `dd`** to GPT labels once any OS is up, or
- USB ethernet gadget + HTTP/`dd` for rootfs iteration (`root=PARTLABEL=rootfs`
  does not care which host built the ext4).

## Vendor boot script (TFTP)

The `boot.script` blob inside the FIT has been **byte-identical** across the
Jan 2024 EVK and Jan 2025 d16 images we looked at
(`sha256: 53235a8734d5154388057072937e5360d839d47e7b135e89a3f4a6e4617b70c5`).

It hard-codes a **USB-gadget / TFTP pair**: device `192.168.96.1`, server
`192.168.96.20`. If that server address exists on a network the board can
reach, U-Boot can **silently TFTP and boot an unsigned image**.

Development knobs (U-Boot env):

- `force_tftpboot=1` — pull FIT from TFTP
- `force_nfsboot=1` — NFS root over the USB gadget
- `fastboot_entry=1` — enter USB fastboot

Treat TFTP boot as a foot-gun on a shared LAN. Prefer flashing the `kernel`
partition or a local FIT.

Rity packages have also shipped **capsule signing material** next to the
images. Do not reuse vendor keys; do not check those files into a public
repo.

## Kernel cmdline

```
root=PARTLABEL=rootfs rootwait
```

Swap only the `rootfs` filesystem as long as that label stays. No boot-script
edit required for a custom ext4 of reasonable size (keep it under the
partition — Rity’s rootfs slot is large; do not assume 10 GB images fit a
smaller custom layout if you shrink it).
