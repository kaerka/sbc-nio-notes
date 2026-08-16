# Hardware quirks (NIO 12L)

Validated on 16 GB / 256 GB UFS units. Dates are 2026 lab notes.

## Serial console

| | |
|---|---|
| Baud | **921600** 8N1 |
| Common USB UART | Silicon Labs CP210x |
| U-Boot `printenv` | `baudrate=921600` |

**115200 produces garbled output.** If the console looks encrypted or
half-readable, fix baud first.

Linux getty on the debug UART is typically `ttyS0`; a 921600 drop-in is needed
if your image defaults to 115200.

## Download / BROM mode

- Hold **DOWNLOAD**, connect **USB‑C OTG** to the host, then release.
- USB ID **`0e8d:0003`** → `/dev/ttyACM*` (MediaTek BROM).
- Tools: `genio-flash` / `mtk-flash` with matching **`lk.bin` (DA)** and **`fip`**
  next to the payload.
- The MT6359 PMIC can power the SoC as soon as USB‑C is connected, so download
  mode is **timing-sensitive**.
- Runtime on OTG: use a **dedicated 5 V supply**. Bus power from a laptop is
  not enough.
- Armbian flash with `mtk-flash`: the **UFS disk image alone is not enough**.
  Matching **`.lk.bin` (DA)** and **`.fip.img`** have to sit next to it.
- USB ethernet gadget (RNDIS / `usb0`) that Rity uses for NFS/TFTP was **not**
  observed on Armbian during bring-up.

## 16 GB RAM overlay

U-Boot `boot_conf` on a d16 board looks like:

```
boot_conf=#conf-mediatek_genio-1200-radxa-nio-12l.dtb#conf-apusys.dtbo#conf-ddr-16g.dtbo#conf-gpu-mali.dtbo#conf-video.dtbo
```

- Base DTB is **`mediatek_genio-1200-radxa-nio-12l.dtb`**, not the EVK DTB.
- **`ddr-16g.dtbo` is required** on 16 GB hardware (do not boot a 8 GB overlay).
- Storage is UFS presented as SCSI (`storage=scsi`, `storage_dev=2`). Linux
  shows well-known LUNs plus a user LUN (`/dev/sdX`, GPT **PARTLABELs**).

## USB host (Type‑A)

Two host stacks:

| Controller | What it drives |
|---|---|
| `xhci-mtk` (`11290000.xhci`) | SoC USB; Wi‑Fi is *not* this, but it shows up as extra buses |
| VIA **VL805** PCIe xHCI | Physical **Type‑A** ports |

### Kernel config (VL805)

Need **`CONFIG_USB_XHCI_PCI=y`**. That option is gated on `USB_XHCI_HCD` and
fights `USB_XHCI_PCI_RENESAS`; the working combo on the Radxa 5.15 tree was:

- `CONFIG_USB_XHCI_HCD=y`
- `CONFIG_USB_XHCI_PCI=y`
- `CONFIG_USB_XHCI_PCI_RENESAS` **disabled** (so PCI xHCI can be built-in)

Also enable **`CONFIG_SND_USB_AUDIO=y`** (plus `SND_HWDEP` / `SND_RAWMIDI`) if
you want USB mics without module chicken-and-egg issues.

### Device tree (mtu3 / Type‑A)

Stock DTS leaves the mtu3 OTG block in `dr_mode = "otg"` with
`usb-role-switch`, waiting on **MT6360 TCPC CC pins**. Type‑A connectors have
no CC pins, so the port never becomes host.

Working patch set:

1. `&ssusb`: `dr_mode = "host"`
2. Remove **`usb-role-switch`** from that node (otherwise the driver still waits
   on TCPC).
3. Enable parent **`&ssusb1`** (`usb@112a1000`) *and* child **`&xhci2`**.
   Enabling only `xhci2` is not enough.

A second `dr_mode = "otg"` on a **different** USB controller is normal — leave
it. After this, Logitech receivers, USB 3.0 sticks, and UAC devices enumerate
on the Type‑A buses.

## Ethernet (`end0`)

Linux names the GbE port **`end0`**, not `eth0`. MAC is `dwmac-mediatek`; PHY
is **RTL8211F**.

On Armbian 6.19 `edge-genio` at **1 Gbps** the link repeatedly:

1. `Link is Up - 1Gbps/Full`
2. `Found uncorrectable error in MAC: 'FSMPES: FSM State Parity Error'`
3. `Reset adapter`

Reproduced with a second cable — not a single bad patch cord; Auto-MDIX is
fine. The cycle **survived package updates**. Treat it as a GMAC/PHY/driver
problem on that edge kernel, not “fix the switch.”

Netplan on the minimal image uses **systemd-networkd** (NetworkManager is
absent). `10-dhcp-all-interfaces.yaml` will **override** a static
`armbian.yaml` on `end0` if you leave the catch-all enabled.

## Wi‑Fi (MT7921)

- Driver: **`mt7921e`**, iface typically `wlp1s0`.
- **Silicon is fine.** The same physical board associated on 5 GHz under
  Ubuntu (NetworkManager) about a week before the Armbian autopsy.
- On Armbian, “no 5 GHz” is almost always a **tooling false negative**:
  - Unprivileged `iw` is often not on `PATH` (`/usr/sbin/iw`); `iw scan` →
    `Operation not permitted`.
  - `iw scan dump` as a normal user returns **only the associated BSS** (so
    one 2.4 GHz row and **zero** 5 GHz).
  - Minimal image has **no NetworkManager / `nmcli` / `nmtui`**. Stack is
    netplan → **networkd + wpa_supplicant**. Status always shows the
    connected 2.4 GHz SSID.
  - `armbian-config` Wi‑Fi runs a **root** `iw … scan` and would list MHz;
    looking only at the associated SSID still fools you.
- Privileged scans on a live 5 GHz neighborhood saw **dozens** of 5 GHz BSS
  (50–65 in one apartment-scale sample, UNII-1/2/3 including DFS).
- **Scan ≠ associate.** We did not complete a 5 GHz STA join on Armbian.
- `txpower` of a few dBm on 2.4 GHz with healthy HE rates is **not** evidence
  that 5 GHz RX is dead.

## APU / NPU (MDLA)

**Not a one-line enable.**

| Stack | `/dev/apusys` | Notes |
|---|---|---|
| Armbian 6.19 `edge-genio` | **missing** | See autopsy below. Panfrost GPU **does** probe. |
| Public Radxa 5.15 (`github.com/radxa/kernel`) | **missing** | Same generation as Rity, but **`CONFIG_MTK_APU` / APUSYS are not in this tree.** |
| MediaTek AIoT BSP | planned path | `gitlab.com/mediatek/aiot/bsp/linux` branch `mtk-v5.15-dev` + `mtk-apusys-driver` + `mtk-apusys-firmware` (`mt8195`) + NeuroPilot / `meta-nn`. |

Rity’s `boot_conf` already includes **`apusys.dtbo`**. That overlay does **not**
create a userspace APU on Armbian or on the public Radxa kernel.

### Armbian 6.19 autopsy (2026-08-02)

Kernel `6.19.8-edge-genio`, `BOARD=radxa-nio-12l`, `BOOT_SOC=mt8395`. This is
**not** a failed probe of a present stack — the stack is not there.

Absent: `/dev/apusys*`, `/dev/mdla*`, `/dev/accel*`, `/dev/vpu*`,
`/dev/neuron*`. No matching `.ko` under `/lib/modules/$(uname -r)`. `lsmod`
has no `apu` / `mdla` / `neuron` / `galcore`. No APU firmware in
`/lib/firmware` (ignore `v4l-cx23418-apu.fw` — name collision).

What *is* in DT, and is **not** a runtime:

- `apu_mem` → `/reserved-memory/memory@62000000` (`shared-dma-pool`)
- `apusys_pll` → `/soc/clock-controller@190f3000`
- Sysfs clocks: `clk-mt8195-apusys_pll`, `clk-mt8365-apu` (platform driver
  **names**)
- `CONFIG_COMMON_CLK_MT8195_APUSYS=y` — **clocks only**
- No `CONFIG_MTK_APU` / APUSYS / MDLA symbols in `/proc/config.gz`
- `armbianEnv.txt`: `fdtfile=mediatek/mt8395-radxa-nio-12l.dtb` (Armbian DTB
  name — different from Rity’s `genio-1200-radxa-nio-12l`)
- **No `apusys` overlay applied**; `/boot/dtb/mediatek/overlay/` was
  empty/missing on this image

Userspace:

- `apt-cache search neuron` hits the biology **NEURON** simulator, not MediaTek
- `libmali-mtk-8195` is **Mali GL** (~19 MB), not APU
- `onnxruntime*` / Arm NN in apt are **CPU** paths
- No NeuroPilot / `neuron_runtime` packages

Do not `apt install` Mali or ONNX expecting `/dev/apusys`. Clock driver names
in sysfs are **not** an inference runtime.

## USB audio (ReSpeaker)

Once Type‑A host + `SND_USB_AUDIO=y` are in:

| Device | ALSA |
|---|---|
| ReSpeaker v2.0 (UAC 1.0) | `card *: ArrayUAC10` |
| ReSpeaker XVF3800 (UAC 2.0) | `card *: Array` after **USB firmware 2.0.7** |

XVF3800 boards can ship with **I2S firmware**. They will not speak UAC until
flashed over **DFU** to the USB audio firmware. Capture as a non-root user
needs membership in **`audio`**. Persist mixer state with `alsactl store` if
`alsa-restore` is in the image.

`ffmpeg` is enough for simple chunked capture if you do not want
`python3-sounddevice`.

## CSI / cameras

Rity DTBOs include `radxa-nio-12l-camera1-imx214` / `camera2-imx214` and a pile
of EVK camera overlays. Apply the ones that match the sensor.

Early Ubuntu/MediaTek image: MTK V4L2 vdec/venc nodes appear in dmesg, but
`media-ctl -p` on `/dev/media0` failed (`ENOENT`). **CSI is not “just
plug in `libcamera`.”** See the raw dump:
[../nio-v4l-csi-ubuntu-mtk.txt](../nio-v4l-csi-ubuntu-mtk.txt).

Need `CONFIG_MEDIA_SUPPORT=y` / `CONFIG_VIDEO_DEV=y` (and the matching DTBO)
before userspace will see a pipeline. Those were still `=m` on an older BSP
config; promoting them to `=y` is what we now bake. Not yet proven on-camera.

## Display (HDMI vs USB‑C DP Alt Mode)

**HDMI works.** On Armbian 6.19, DRM shows **`card0-HDMI-A-1`** only. A
linuxkms / GBM UI (Slint, etc.) can take over that connector with no desktop.

USB‑C **DP 1.4 Alt Mode** is real hardware (Radxa docs). Path on the board:

```
USB‑C receptacle
  ├─ USB HS/SS  → MTU3 (OTG)
  ├─ SBU / orientation → ITE IT5205 typec-mux @ 0x48
  ├─ PD / CC     → MT6360 TCPC (PMIC @ 0x34)
  └─ DP lanes    → SoC DP‑TX  (when Alt Mode + mux select DP)
```

On Armbian 6.19 the **pieces exist but the pipeline is off**:

| Present | Broken / disabled |
|---|---|
| DT: `mediatek,mt6360-tcpc`, `usb-c-connector`, `ite,it5205` | `mt6360-tcpc: Failed to register tcpci port` |
| Modules: `tcpci_mt6360`, `tcpm`, `typec`, `it5205` | MT6360 regulator / device-link failures (`regulator-vsys-buck`) |
| `typec_displayport.ko`, `mtk_dp.ko` available | `dp-tx@1c600000` (`mediatek,mt8195-dp-tx`) **`status = "disabled"`** |
| HDMI `hdmi-tx` okay | `dp-intf@1c113000` **`status = "disabled"`** |
| | No Type‑C partner; `orientation=unknown`; no `DP-*` DRM connector |

This is DT + TCPC bring-up, not `apt install`. Enable `dp-tx` / `dp-intf`,
graph-link them into MediaTek DRM **and** the USB‑C Alt Mode port, and get
`/sys/class/typec/port0` to show a partner. Collabora’s 2024 NIO DT work
already noted USB role switching **without** Alternate Modes
([linux-arm-kernel](https://lists.infradead.org/pipermail/linux-arm-kernel/2024-April/918578.html)).

A USB‑C accessory can still enumerate as **USB 2.0 HID only** (we saw XR
glasses do exactly that). That is **not** a DP Alt Mode test — use a known DP
dock/monitor.
