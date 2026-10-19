# TTL flashing procedure

Stock RG-MA3063 units have **no web U-Boot** — recovery/flashing is done over
the UART (TTL) console. This is the community-proven procedure.

## ⚠️ Before you start

1. **Check the flash vendor** in the U-Boot banner
   (`Serial NAND device Manufacturer:...`). If it is Fudan Micro (复旦微),
   **do not flash** — see [01-hardware-and-research.md](01-hardware-and-research.md).
2. **Back up** the stock firmware first (see the backup section below). At a
   minimum dump the `ART` partition — you need it for the Wi-Fi calibration.
3. Expect the stock **rollback guard** to fight you: flash **both** rootfs
   slots in one go. If the device keeps reverting to stock, restore the stock
   backup, boot the official firmware once, then flash both slots again
   (documented workaround in the community thread).

## Wiring

* UART: TTL 115200, 8N1, on the console header (U-Boot console is
  `serial@78AF000` = `blsp1_uart1`).
* Ethernet: the cable for TFTP must be plugged into the port **nearest the
  power barrel** (this is the only port that answers U-Boot TFTP).
* Host: static IP `192.168.1.123`, TFTP server (e.g. TFTPD64) serving the
  firmware file.

## Procedure

```sh
# 1. Interrupt autoboot (press Enter repeatedly during "Hit any key to stop
#    autoboot") until you get the prompt:
#    IPQ5018#

# 2. Configure network
setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.123
saveenv

# 3. Copy the firmware to the TFTP root as EXACTLY "op.bin" (no typos!)
#    The factory image: openwrt-qualcommax-ipq50xx-ruijie_rg-ma3063-squashfs-factory.ubi

# 4. Flash both slots
tftpboot op.bin && flash rootfs && flash rootfs_1
```

U-Boot's `flash rootfs` maps to the SMEM partition `rootfs`
(`0x3b00000`, 50 MiB); `flash rootfs_1` writes the backup slot
(`0x900000`, 50 MiB).

## Backing up stock firmware (do this before flashing anything)

```sh
# At the IPQ5018# prompt, after setting ipaddr/serverip as above:
# e.g. read 1 MiB of ART to RAM and write to TFTP
# (commands vary by QSDK build; "nand info" is NOT available on this build,
#  use the smem partition table shown by the "smem" command)

setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.123
nand read 0x44000000 0x780000 0x100000 && tftpboot 0x44000000
# (or use "nand dump" equivalents supported by this U-Boot)
```

Store the dumps somewhere safe. The **ART** dump is the most valuable: it
holds the MAC addresses and the Wi-Fi calibration used to build the
`ipq-wifi` board file (see [06-ipq-wifi-bdf.md](06-ipq-wifi-bdf.md)).

> The exact `nand read`/`tftpboot` invocation differs between QSDK builds —
> verify with `help` at the prompt. Document your working commands in
> [05-bringup-checklist.md](05-bringup-checklist.md) once verified on the
> bench.

## First boot after flashing

* Console: 115200 8N1 on `blsp1_uart1`.
* The FIT kernel (UBI `fit` volume) is booted by `bootipq`.
* Expected interfaces: `eth0` (gmac0), `lan1`/`lan2`/`lan3`/`wan`
  (DSA on gmac1 via the QCA8337).
* OpenWrt default: LAN side DHCP at `192.168.1.1`.
