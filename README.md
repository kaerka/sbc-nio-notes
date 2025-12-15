# sbc-nio-notes
This repository documents real-world experimentation with the Radxa NIO SBC.

Goals:
- Portable desktop
- Edge AI / NPU experimentation
- Camera feasibility on Armbian

Non-goals:
- Chasing vendor-only blobs indefinitely
- Month-long kernel archaeology
- Treating this as a Raspberry Pi replacement

Device Specs:
- 8c arm64 processor (big+little config)
- 16gb ram
- 256gb onboard UFS storage
- No eMMC storage
- 1gb ethernet
- Onboard PCIe mediatek wifi (2.4 + 5ghz)
- hdmi output
- hdmi input (but this seems to operate as a pass-through)
- CSI 4-lane MIPI camera interface (dual, there is a 2nd one on the underside of the board)

> The only serious drawback on this that I see - is the 256gb storage, and not having a PCIe NVMe SSD option.