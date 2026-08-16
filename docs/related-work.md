# Related work (other NIO 12L notes)

We did **not** copy text, DTS, or scripts from elsewhere. Lessons listed here were
learned by [Zach Lorenzini](https://github.com/zlorenzini/radxa-nio-12l)
([zlorenzini/radxa-nio-12l](https://github.com/zlorenzini/radxa-nio-12l),
Armbian 6.19 + earlier Ubuntu/Rity). If you use a fix, go read **their** docs
and apply from that repo.

We have **not** reproduced all of this on our boards.

## Overlap (we already hit this)

- Dedicated 5 V supply; laptop USB-C is not enough
- Hold DOWNLOAD, USB-C OTG, BROM `0e8d:0003`
- 8 GB vs 16 GB firmware must match (`ddr16g` / `ddr-16g.dtbo`)
- HDMI works; USB-C TCPC / `mt6360-tcpc` fails to register on Armbian 6.19
- APU not actually usable on stock Armbian; NeuroPilot is an Ubuntu/Genio path

## New to us (worth tracking)

### Power

- They specify **5 V / 3 A minimum**, **5 V / 5 A preferred**, **right-side
  USB-C** as the power jack.
- **USB PD negotiation is not the way to feed this board.** A PD brick that
  only raises current after handshake can sit at ~0.9 A @ 5 V and look like a
  firmware boot loop.
- Under-current: boots, then resets in kernel/userspace forever.

### GPU (Panfrost on 6.19)

- Stock **`gpu-mali.dtbo`** can wire `mali_sram-supply` to the **wrong PMIC
  node** (`mt6359_vsram_others_ldo_reg` instead of `mt6315_7_vbuck1`). Mali
  then logs `Couldn't update frequency transition information` and **sticks at
  390 MHz** (no DVFS). Their fix lives on the firmware partition DTB overlay;
  see their `docs/gpu-acceleration.md`.
- Long-running EGL (browser) on Mesa 25.x + Panfrost: `JOB_STATUS_INVALID_DATA_FAULT`.
  **`panfrost.no_afbc=1` is ignored on 6.19.** Mesa `PAN_MESA_DEBUG=noafbc` is
  what they used. Watchdog resets after hours can be this, not “random Armbian.”
- `glmark2-es2-wayland` **2699** after DVFS + noafbc (their bench).
- Vulkan/`panvk` for Bifrost: not in Mesa 25.2.

### MT6360 / IRQ 116 (same TCPC hole we saw)

- Boot ~54 s: failed `mt6360` regulator register, failed device-link to
  `regulator-vsys-buck`, `Failed to register tcpci port`, then **`irq 116:
  nobody cared`** and the kernel **disables** that IRQ.
- They blacklisted `tcpci_mt6360` so the storm stops. USB-C PD was already
  dead. Upstream DTS work they point at is the Kontron I1200 MT8395 series
  (LKML, Jul 2025) — same PMIC family.
- This matches our “TCPC probe fails, `dp-tx` disabled” Armbian notes, with a
  concrete interrupt-storm mechanism we had not written down.

### CMA / video DMA

- `cma=4096M` **fails to reserve** on this DRAM map (`CmaTotal=0`). They
  settled on **`cma=256M`** plus **`swiotlb=262144`** in `armbianEnv.txt`
  `extraargs` after IOMMU/`swiotlb buffer is full` during video/DMA.

### Ubuntu / Genio (not Armbian)

- HDMI DDC can fail if the monitor is hot-plugged late
  (`mediatek-hdmi-mt8195-ddc: ddc failed`). Plug HDMI **before** power.
- GDM/GNOME without a display: crash-respawn loop; disable GDM for headless.
- Proprietary `libmali-mtk-8195` lacks `EGL_KHR_image_pixmap`; Glamor/Xorg
  dies. They used `AccelMethod none` or an EGL preload shim. **Armbian
  Panfrost does not need that shim.**
- NeuroPilot packages exist on **Ubuntu Jammy + `ppa:mediatek-genio/genio-public`**
  (`mediatek-apusys-firmware-genio1200`, `mediatek-libneuron`, …). Their
  README still marks NPU as untested on the Armbian track.

### Flash host extras

- Extra BROM ID **`0e8d:201c`** besides `0e8d:0003`; udev `plugdev` rules for
  both (and FTDI `0403`) in their `docs/hardware.md`.
- Green LED pulse = download transfer.

### Specs they list vs ours

Treat as **theirs**, not gospel: they write **LPDDR4X** and **UFS 2.1**. Our
Rity/Radxa notes said **LPDDR5** and **UFS 3.1** (Samsung `KLUEG8U1EA-B0C1`).
Do not “fix” our docs to match until someone reads `dmidecode`/UFS inquiry on
a live board.

## What we still have that they do not cover

- Serial **921600** (easy to miss)
- Named UFS partitions / FIT wrap / TFTP `192.168.96.1`/`20` foot-gun
- Type‑A host DTS (`dr_mode`, drop `usb-role-switch`, `ssusb1`/`xhci2`) + VL805
  `CONFIG_USB_XHCI_PCI`
- `end0` **FSMPES** GbE reset loop on Armbian
- 5 GHz Wi‑Fi scan false negatives (`mt7921e`)
- Public Radxa 5.15 Yocto kernel / `argus-image-full` path
