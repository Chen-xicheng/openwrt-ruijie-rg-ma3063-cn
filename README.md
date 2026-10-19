# OpenWrt support for the Ruijie RG-MA3063 (IPQ5018)

OpenWrt (Qualcommax / ipq50xx) support for the **Ruijie RG-MA3063** — a dual-band
AX3000 router based on the Qualcomm IPQ5018 SoC. This project tracks the
bring-up effort: device tree, image build configuration, boot research, TTL
flashing procedure and on-device bring-up verification.

> **STATUS: DEVELOPMENT — Fully Stable.** 
> All function tested
> Wifi 2.4Ghz and 5Ghz both can running
> Ethernet is UP
## ⚠️ Read this first — flash variants

The RG-MA3063 was shipped with (at least) three different SPI-NAND chips:

| Flash | Status |
|---|---|
| Macronix MX35UF1GE4AC (ID `c2 92`) | OK — this is what the referenced U-Boot log shows |
| Winbond W25N01GWZEIG | Likely OK |
| **Fudan Micro (复旦微)** | **DO NOT FLASH.** Community reports: units with Fudan Micro flash fail to flash custom firmware, boot-loop and can brick (`复旦微的闪存勿刷`) |

Before flashing anything: watch the first U-Boot line mentioning
`Serial NAND device Manufacturer:` — if it is **not** a supported vendor,
stop. See [docs/01-hardware-and-research.md](docs/01-hardware-and-research.md).


```

## The changes (patch summary)

The single patch adds one device to the Qualcommax `ipq50xx` subtarget:

1. `target/linux/qualcommax/dts/ipq5018-rg-ma3063.dts` — device tree
   (model `ruijie,rg-ma3063`, serial console `blsp1_uart1`, QCA8337 switch
   with 3x LAN + 1x WAN, IPQ5018 2.4G + QCN6102 5G radios, ART nvmem MACs).
2. `target/linux/qualcommax/image/ipq50xx.mk` — `Device/ruijie_rg-ma3063`
   entry: `UbiFit`, 128k blocksize, 2048 page, 51200k rootfs image, NAND 128m.
3. `target/linux/qualcommax/ipq50xx/base-files/etc/hotplug.d/firmware/11-ath11k-caldata`
   — calibration extraction case for `ruijie,rg-ma3063` (mirrors the sibling
   `cmcc,mr3000d-ci`).

Apply it against `openwrt/openwrt` at commit
`486b4a4b3ede480635a185234fc624c969426f13` (2026-08-09):

```sh
git -C openwrt apply patches/0001-qualcommax-ipq50xx-add-ruijie-rg-ma3063.patch
```

See [docs/03-build-instructions.md](docs/03-build-instructions.md) for the full
build, and [docs/04-ttl-flash.md](docs/04-ttl-flash.md) for flashing.

## Why ipq50xx

* SoC: Qualcomm **IPQ5018** (dual-core A53, 256MB RAM)
* Wi-Fi: IPQ5018 internal **2.4 GHz** + **QCN6102** 5 GHz over PCIe
  (same radio family as the sibling devices, so the `ipq5018-qcn6122.dtsi`
  blob applies)
* Switch: **QCA8337** (QCA8337-AL3C) — 4 GbE user ports (1 WAN + 3 LAN)
* Storage: 128MB SPI-NAND in two rootfs slots (`rootfs` + `rootfs_1`),
  flashed via U-Boot TTL

## Related

* Reference board files in the same OpenWrt subtarget: `ipq5018-mr3000d-ci.dts`,
  `ipq5018-pz-l8.dts`
* acwifi teardown (Chinese): https://www.acwifi.net/21472.html
* Community teardown / flash write-up (Chinese):
  https://godsun.pro/blog/rui-jie-rg-ma3063.html
* Efxn teardown of the Winbond-flash variant (Chinese):
  https://www.efxn.net.cn/threads/锐捷-rg-ma3063-ax3000-路由器-移动定制版拆解.47/
* SSH unlock methods (factory/developer mode): https://ioozu.com/post-3541.html
* QSDK12 custom firmware thread with the Fudan Micro flash warning.

## License

The device tree carries `GPL-2.0-or-later OR MIT` (same as the sibling files).
Documentation is CC0 unless stated otherwise.
