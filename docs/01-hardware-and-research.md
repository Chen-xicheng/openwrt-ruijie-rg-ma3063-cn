# Hardware & research

Facts come from the documented U-Boot TTL session and the stock OpenWrt
`/proc/mtd` dump, cross-referenced against the sibling boards CMCC MR3000D-CI
and CMCC PZ-L8, and the acwifi / Efxn teardowns (see Sources).

## Verified hardware

| Item | Value |
|---|---|
| SoC | Qualcomm **IPQ5000** silicon (silkscreen confirms, acwifi teardown photo; dual Cortex-A53 @1GHz, `machid 8040000`). The QSDK U-Boot prompt/platform identifies as **IPQ5018** — same platform family, one shared firmware for all revisions. Mainline has no separate `ipq5000.dtsi`; OpenWrt's `ipq50xx` subtarget builds these boards on `ipq5018.dtsi` |
| RAM | **256 MB** (Winbond DDR3L; `free -m` → ~189MB usable). Note: some units place the SoC + RAM on the back of the PCB |
| Flash | 128MB SPI-NAND (see variant warning below) |
| 2.4 GHz radio | IPQ5018 internal (`wifi@c000000`) + FEM **KCT8239S** |
| 5 GHz radio | QCN6102 2x2 @160MHz on PCIe (`wifi@b00a040`) + FEM **KCT8539S** |
| Switch | QCA8337 (QCA8337-AL3C), 4x GbE user ports (1 WAN + 3 LAN), auto-MDI/MDIX ("盲插") |
| U-Boot | QSDK 2016.01 (May 19 2023), console `serial@78AF000` = `blsp1_uart1` @ 115200 |
| Boot | `bootcmd=bootipq` — boots the `fit` volume of the `rootfs` UBI |
| MAC | ART offset 0x0 (MAC0) and 0x6 (MAC1); on the logged unit both equal |
| PSU | 12V/1A, ~5.8W standby |
| Notes | Default config is **bridge mode** (not router); supports 802.11k/v only when mesh is enabled; beamforming yes, MU-MIMO no |

> SoC marking: the silicon is etched **IPQ5000** (confirmed from the acwifi
> teardown photo). The QSDK platform (U-Boot `IPQ5018#` prompt, machid) and the
> OpenWrt subtarget (`SOC := ipq5018`) treat IPQ5000 and IPQ5018 as one
> platform — Ruijie ships the same firmware for every revision, which is why
> the `ipq5018.dtsi`-based device tree applies.

## ⚠️ Flash & hardware variant warning

Two published teardowns and a community thread show the MA3063 was shipped
with different parts across hardware revisions:

| Revision evidence | Flash | RAM | PCB layout |
|---|---|---|---|
| acwifi teardown | Macronix **MX35UF1GE4AC** (ID `c2 92`) | 256MB | SoC on main face |
| Efxn teardown (2026-01-11) | Winbond **W25N01GWZEIG** | Winbond DDR3L 256MB | SoC + RAM on the back |
| community (godsun QSDK12 thread) | **Fudan Micro (复旦微)** | — | — |

Units with **Fudan Micro** flash are essentially unflashable with custom
firmware — boot loops, rollback to stock and bricks. The QSDK12 thread
literally says *"复旦微的闪存勿刷"* (*"Fudan Micro flash: do not flash"*).

U-Boot prints the part at every boot:

```
NAND: QPIC controller support serial NAND
ID = 192c2
Vendor = c2
Device = 92
Serial NAND device Manufacturer:MX35UF1GE4AC   <-- check this line
```

Safe identifiers seen: `c2 92` (Macronix MX35UF1GE4AC), Winbond
W25N01GWZEIG. If the printed vendor is Fudan Micro, do **not** attempt a flash.

## Sources

* acwifi teardown (Chinese): https://www.acwifi.net/21472.html
  (mirror: https://www.smyz.net/luyouqi/16807.html)
* Efxn teardown (Chinese, 2026-01-11, Winbond variant):
  https://www.efxn.net.cn/threads/锐捷-rg-ma3063-ax3000-路由器-移动定制版拆解.47/
* godsun teardown + TTL boot log + flashing:
  https://godsun.pro/blog/rui-jie-rg-ma3063.html
* SSH unlock methods: https://ioozu.com/post-3541.html

## Partition table (from U-Boot `smem` / stock `/proc/mtd`)

```
No.: Name         Start      Size
0:   0:SBL1      0x000000   0x80000
1:   0:MIBIB     0x080000   0x80000
2:   0:BOOTCONFIG 0x100000  0x40000
3:   0:BOOTCONFIG1 0x140000 0x40000
4:   0:QSEE      0x180000   0x100000
5:   0:QSEE_1    0x280000   0x100000
6:   0:DEVCFG    0x380000   0x40000
7:   0:DEVCFG_1  0x3c0000   0x40000
8:   0:CDT       0x400000   0x40000
9:   0:CDT_1     0x440000   0x40000
10:  0:APPSBLENV 0x480000   0x80000
11:  0:APPSBL    0x500000   0x140000
12:  0:APPSBL_1  0x640000   0x140000
13:  0:ART       0x780000   0x100000   <- MACs + Wi-Fi calibration
14:  0:TRAINING  0x880000   0x80000
15:  rootfs      0x3b00000  0x3200000  <- 50 MB UBI (boot slot A)
16:  rootfs_1    0x900000   0x3200000  <- 50 MB UBI (boot slot B)
17:  ttyMTD      0x6d00000  0x120000
18:  productinfo 0x6e20000  0x80000
19:  data        0x6ea0000  0x1040000
```

`rootfs` UBI geometry (from U-Boot attach log) — **matches the OpenWrt
`UbiFit` factory.ubi exactly**:

```
PEB size: 131072 (128KiB), LEB size: 126976
min/max I/O: 2048/2048, sub-page 2048
VID header offset: 2048, data offset: 4096
```

## Stock U-Boot environment (reference)

```
baudrate=115200
bootargs=console=ttyMSM0,115200n8
bootcmd=bootipq
fdt_high=0x4A400000
mtdparts=mtdparts=nand0:0x3200000@0x3b00000(fs)
ethaddr=9c:2b:a6:1b:85:51
eth1addr=9c:2b:a6:1b:85:51
```

## Ethernet topology

Both sibling reference boards wire the IPQ5018 the same way:

```
IPQ5018 gmac0  --- internal GE PHY --- QCA8337 Phy4
IPQ5018 gmac1  --- SGMII (uniphy0) --- QCA8337 CPU port
```

The MA3063 DTS follows the CMCC PZ-L8 topology (SGMII CPU port on switch
port 0 + gmac0 uplink on switch port 5), which is the layout the MA3063
community firmware is based on. **The exact CPU port (0 vs 6) is listed as a
bring-up verification item** — see the checklist.
