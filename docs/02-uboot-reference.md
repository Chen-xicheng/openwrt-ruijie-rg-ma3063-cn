# U-Boot boot log (annotated)

Reproduced from the community teardown write-up
(https://godsun.pro/blog/rui-jie-rg-ma3063.html). This is the reference for
the device tree and the flashing procedure.

## Boot output

```
U-Boot 2016.01 (May 19 2023 - 11:39:14 +0800)

DRAM: smem ram ptable found: ver: 1 len: 4  256 MiB
NAND: QPIC controller support serial NAND
ID = 192c2
Vendor = c2
Device = 92
Serial Nand Device Found With ID : 0xc2 0x92
Serial NAND device Manufacturer:MX35UF1GE4AC
Device Size:128 MiB, Page size:2048, Spare Size:64, ECC:4-bit

SF: Unsupported flash IDs: manuf 00, jedec 0000, ext_jedec 0000
ipq_spi: SPI Flash not found (bus/cs/speed/mode) = (0/0/48000000/0)

128 MiB
PCI0 is not defined in the device tree
PCI1 is not defined in the device tree

In: serial@78AF000
Out: serial@78AF000
Err: serial@78AF000
machid: 8040000
Hit any key to stop autoboot: 0
0
Net: cmbblk is stable 5
MAC0 addr:9c:2b:a6:1b:85:51
PHY ID1: 0x4d
PHY ID2: 0xd0c0
MAC0 addr:9c:2b:a6:1b:85:51
GMAC1:Invalid PHY ID
eth0, eth1
IPQ5018#
```

## UBI attach log (from the `smem` command)

```
ubi0: attaching mtd1
ubi0: attached mtd1 (name "mtd=0", size 50 MiB)
ubi0: PEB size: 131072 bytes (128 KiB), LEB size: 126976 bytes
ubi0: min./max. I/O unit sizes: 2048/2048, sub-page size 2048
ubi0: VID header offset: 2048 (aligned 2048), data offset: 4096
ubi0: good PEBs: 400, bad PEBs: 0, corrupted PEBs: 0
ubi0: user volume: 5, internal volumes: 1, max. volumes count: 128
```

The 50 MiB "mtd1" UBI is the `rootfs` partition. Stock UBI volumes:
`kernel`, `wifi_fw`, `bt_fw`, `ubi_rootfs`, `rootfs_data`.

## `smem` partition table

```
No.: Name         Attributes Start     Size
0:   0:SBL1      0x0000ffff 0x0       0x80000
1:   0:MIBIB     0x0000ffff 0x80000   0x80000
2:   0:BOOTCONFIG 0x0000ffff 0x100000 0x40000
3:   0:BOOTCONFIG1 0x0000ffff 0x140000 0x40000
4:   0:QSEE      0x0000ffff 0x180000   0x100000
5:   0:QSEE_1    0x0000ffff 0x280000   0x100000
6:   0:DEVCFG    0x0000ffff 0x380000   0x40000
7:   0:DEVCFG_1  0x0000ffff 0x3c0000   0x40000
8:   0:CDT       0x0000ffff 0x400000   0x40000
9:   0:CDT_1     0x0000ffff 0x440000   0x40000
10:  0:APPSBLENV 0x0000ffff 0x480000   0x80000
11:  0:APPSBL    0x0000ffff 0x500000   0x140000
12:  0:APPSBL_1  0x0000ffff 0x640000   0x140000
13:  0:ART       0x0000ffff 0x780000   0x100000
14:  0:TRAINING  0x0000ffff 0x880000   0x80000
15:  rootfs     0x0000ffff 0x3b00000   0x3200000
16:  rootfs_1   0x0000ffff 0x900000    0x3200000
17:  ttyMTD     0x0000ffff 0x6d00000   0x120000
18:  productinfo 0x0000ffff 0x6e20000  0x80000
19:  data       0x0000ffff 0x6ea0000   0x1040000
```

## `printenv`

```
baudrate=115200
bootargs=console=ttyMSM0,115200n8
bootcmd=bootipq
bootdelay=1
eth1addr=9c:2b:a6:1b:85:51
ethact=eth0
ethaddr=9c:2b:a6:1b:85:51
fdt_high=0x4A400000
fdtcontroladdr=4a984004
flash_type=11
ipaddr=192.168.10.10
ipqboot_num=0
machid=8040000
mtddevname=fs
mtddevnum=0
mtdids=nand0=nand0
mtdparts=mtdparts=nand0:0x3200000@0x3b00000(fs),
netmask=255.255.255.0
partition=nand0,0
serverip=192.168.10.19
soc_hw_version=20180101
soc_version_major=1
soc_version_minor=1
stderr=serial@78AF000
stdin=serial@78AF000
stdout=serial@78AF000
```

## Stock firmware `/proc/mtd` (post-UBI)

```
mtd15: 03200000 00020000 "rootfs_1"
mtd16: 03200000 00020000 "rootfs"
mtd17: 00120000 00020000 "ttyMTD"
mtd18: 00080000 00020000 "productinfo"
mtd19: 01040000 00020000 "data"
mtd20: 003dc008 0001f000 "kernel"        <- UBI volume kernel
mtd21: 0036b000 0001f000 "wifi_fw"       <- UBI volume wifi_fw
mtd22: 00004800 0001f000 "bt_fw"         <- UBI volume bt_fw
mtd23: 00fdd000 0001f000 "ubi_rootfs"    <- UBI volume ubi_rootfs
mtd24: 01132000 0001f000 "rootfs_data"   <- UBI volume rootfs_data
```

## Notes

* `bootipq` boots the UBI `fit` volume in the `rootfs` partition — the exact
  format OpenWrt's `Device/UbiFit` factory.ubi provides.
* There is **no web U-Boot** on stock units; flashing is TTL-only.
* Stock firmware has a **rollback guard**: one user reported boot-looping back
  to stock after flashing a single slot. Always flash **both** `rootfs` and
  `rootfs_1`.
