---
title: Calix GP1100G
has_children: false
parent: Calix
---

# Hardware Specifications

|                  |                                                        |
| ---------------- | ------------------------------------------------------ |
| Vendor/Brand     | Calix                                                  |
| Model            | GP1100G                                                |
| ODM              | Calix                                                  |
| Chipset          | MaxLinear/Lantiq PRX300 (interAptiv MIPS)              |
| Flash            | 256 MB NAND (Winbond, `0xef 0xaa`)                     |
| RAM              | 256 MB (255 MiB usable)                                |
| Bootloader       | U-Boot 2016.07-CALIX-v3.1.311                          |
| System           | Calix EXOS (modified OpenWrt), BusyBox v1.33.1         |
| 2.5GBaseT        | Yes (2.5 Gbps GPON variant of the GP1100)             |
| Optics           | SC/APC                                                 |
| IP address       | 192.168.100.100/24 (LAN side)                          |
| Web Gui          | ✅ user `admin`, password `fiber1ttx` (killed when fiber is connected) |
| SSH              | ✅ user `support`, password `support`, port `30007` (dropbear) |
| Telnet           | ✅ user `support`, password `support`, port `23`       |
| Serial           | ✅ user `support`, password `support`                  |
| Serial baud      | 115200                                                 |
| Serial encoding  | 8-N-1                                                  |
| Form Factor      | ONT                                                    |

CLEI `BVML800BRB`, Calix part number `3000305811`. Firmware tested:
Calix EXOS-R24.4.500, build `24.4.500.348`.

# Serial

The board has a 3.3V TTL UART header (Lantiq console, 115200 8-N-1). It is the
only recovery path if a bad NVRAM write fails the boot-time CRC check (see
[Recovery](#recovery-from-a-bad-nvram-write)).

::: warning Note
The bootloader does not stop on Enter. During the 5-second autoboot countdown,
type `stop` to drop to the `PRX300 #` U-Boot prompt.
:::

The UART header is the 4-pin block on the right edge of the board, next to the
RJ45 cage (see [Teardown](#teardown-and-other-photos)).

::: warning Note
The TX/RX assignment on the header is not yet confirmed. The port is 3.3V TTL:
if you get no output, swap TX and RX and retry. Do not connect the VCC pin.
:::

<!-- TODO: add a labelled UART pinout close-up (GND / TX / RX / VCC) -->

# Root procedure

The EXOS firmware starts dropbear (SSH) on port `30007` and telnet on port `23`
at boot. The OLT-pushed firewall blocks both on IPv4. The `ip6tables` kernel
module is never loaded, so IPv6 traffic bypasses the firewall. Any device on the
same L2 segment reaches a root shell over the ONT's IPv6 link-local address. No
UART and no firmware modification are needed.

The same bypass works on other Calix EXOS devices (the 100-05609 SFP and the
GigaSpire u6x). Credentials are the hardcoded `support` / `support`.

::: warning Note
The ONT's IPv6 link-local address is **not** derived from the LAN MAC via EUI-64.
It comes from a separate internal CPU MAC, and the GP1100G generates a new SLAAC
address on every boot. Read it from the host's IPv6 neighbor cache rather than
computing it.
:::

Procedure (Windows host, replace the interface index with yours):

```powershell
# 1. Find the NIC connected to the ONT
Get-NetAdapter | Format-Table Name, InterfaceIndex, Status, MacAddress

# 2. Trigger neighbor discovery, then read the ONT's link-local address
ping -6 ff02::1%<ifindex> -n 4
Get-NetNeighbor -InterfaceIndex <ifindex> -AddressFamily IPv6 |
  Format-Table IPAddress, LinkLayerAddress, State
```

```sh
# 3. SSH in (password: support)
ssh -p 30007 support@fe80::<ont-link-local>%<ifindex>
```

::: info Note
SSH and telnet die when the fiber connects. EXOS tears down LAN-side management
when the PON link comes up, and it does not recover when the fiber is pulled. A
reboot restores management access. Persist any logging to `/no_erase/` (UBIFS,
survives reboot) before connecting the fiber.
:::

The web UI uses `admin` / `fiber1ttx`, but EXOS kills the httpd whenever the
fiber is connected.

# List of partitions

NAND layout (`/proc/mtd`):

| dev   | size       | name          | purpose                 |
| ----- | ---------- | ------------- | ----------------------- |
| mtd0  | 0x00100000 | uboot         | bootloader              |
| mtd1  | 0x00040000 | ubootconfigA  | U-Boot environment A    |
| mtd2  | 0x00040000 | ubootconfigB  | U-Boot environment B    |
| mtd3  | 0x00040000 | gphyfirmware  | GPHY firmware           |
| mtd4  | 0x00100000 | calibration   | calibration             |
| mtd5  | 0x01000000 | bootcore      | boot core               |
| mtd6  | 0x06d40000 | system_sw     | system software         |
| mtd7  | 0x008c0000 | no_erase      | persistent (UBIFS)      |
| mtd8  | 0x07200000 | exa_data      | EXA data                |
| mtd9  | 0x00200000 | CSSS          | security store          |
| mtd10 | 0x00080000 | nvram         | **identity (see below)** |
| mtd11 | 0x002c0000 | res           | reserved                |
| mtd12 | 0x0022c760 | kernelA       | kernel A                |
| mtd13 | 0x0085b000 | rootfsA       | rootfs A                |
| mtd14 | 0x00287000 | kernelB       | kernel B                |
| mtd15 | 0x01439000 | rootfsB       | rootfs B                |
| mtd16 | 0x0101b000 | rootfs_data   | overlay                 |
| mtd17 | 0x06d78000 | exa_data_     | EXA data (B)            |
| mtd18 | 0x00744000 | no_erase_     | persistent (B)          |

The device is dual-bank (A/B); U-Boot `active_bank` / `commit_bank` select the
running image.

# GPON ONU status

## Identity storage (NVRAM)

The GPON identity is baked into the `nvram` partition (`mtd10`, 512 KB) as
fixed-width, null-padded fields. The partition is CRC-protected: the first four
bytes are a CRC32 over the payload. A raw write that does not recompute the CRC
fails a boot-time check and bricks the device (recoverable via UART only).

Field layout (first ~0xB0 bytes; example values, real identity redacted):

| Offset | Size | Field              | Example           |
| ------ | ---- | ------------------ | ----------------- |
| 0x00   | 4    | CRC32 of payload   | (computed)        |
| 0x0C   | 16   | Model              | `GP1100G`         |
| 0x1C   | 12   | CLEI               | `BVML800BRB`      |
| 0x28   | 20   | Mfg serial number  | `xxxxxxxxxxxx`    |
| 0x3C   | 12   | Part number        | `3000305811`      |
| 0x48   | 12   | Mfg date           | `MM/DD/YYYY`      |
| 0x5C   | 1    | MAC count          | `05`              |
| 0x5D   | 6    | Base MAC           | `xx:xx:xx:xx:xx:xx` |
| 0x63   | 12   | FSAN / GPON SN     | `CXNKxxxxxxxx`    |
| 0x7C   | 4    | Country code       | `US`              |
| 0x80   | 32   | Firmware build     | `24.4.500.348`    |
| 0xA0   | 12   | HW part number     | `1000580511`      |

The MAC-count byte (`0x5C`) declares a pool of five MACs starting at the base
MAC. The same values also live in `/etc/config/startup-config.xml`
(`<base-mac>`, `<serial-number>`, `<mfg-serial-number>`, DHCP option 125
vi-encap fields, and the TR-069 username keyed to the serial number). NVRAM is
the source of truth; `libpalbsp.so` regenerates the XML from NVRAM when the XML
is missing or corrupt.

## Reading and changing the GPON serial number

`libpalbsp.so` exposes the NVRAM through sysfs and handles the CRC/header on
write. **Use this interface, not raw `dd`/`flashcp` to `mtd10`.**

```sh
# All identity fields
for f in /sys/class/pal_bsp/pal_bsp/nvram/clx_*; do
  echo "$(basename "$f"): $(cat "$f")"
done

# Read / change only the FSAN serial number
cat /sys/class/pal_bsp/pal_bsp/nvram/clx_gpon_sn
echo "CXNKxxxxxxxx" > /sys/class/pal_bsp/pal_bsp/nvram/clx_gpon_sn

# Confirm the CRC (first 4 bytes of mtd10) changed, then reboot
hexdump -C /dev/mtd10 | head -1
reboot
```

sysfs field names include `clx_gpon_sn`, `clx_gpon_pw`, `clx_base_mac`,
`clx_mac_num`, `clx_mfg_sn`, `clx_100_part_number`, `clx_hw_part_number`,
`clx_clei`, `clx_country_code`, `clx_fw_ver`, and `clx_module_name`.

::: warning Change the FSAN serial only
The base MAC seeds a pool of five MACs used by the LAN, PON, and management
interfaces, plus DHCP vi-encap OUI fields and TR-069. The Calix E7 OLT
authenticates on the FSAN serial, not the MAC, so leave the base MAC alone.
:::

## OLT authentication

On the Calix E7-2 (AXOS) plant tested, a matching FSAN serial number alone
reaches `O5` and receives full OMCI provisioning. The ONT reached `O5` with an
empty PLOAM password and no Registration ID. The operator provisions each ONT
by hand in Calix SMx, keyed to the FSAN serial, so the serial is the only link
key and RONTA is not used.

This is plant-dependent. A Calix plant can instead require a PLOAM password, or
activate ONTs through RONTA (Remote ONT Activation) with a Registration ID. The
Registration ID is a separate linking value, not the same thing as the PLOAM
password. On plants that do use a PLOAM password, the Registration ID is
typically the value programmed into the PLOAM password field (its ASCII bytes),
so the two are set together even though they are distinct concepts. If your OLT
gates on them, set them through `clx_gpon_pw` or the Calix Smart Activate UI.

After linking, the OLT pushed the MIB reset/upload, ONU-G / ONU2-G / Software
Image queries, T-CONT and GEM provisioning, and Calix-proprietary ME 241
configuration. The stock EXOS firmware answered every ME correctly.

# Recovery from a bad NVRAM write

A raw write to `mtd10` that leaves the CRC stale fails this boot check:

```
[calix_read_nvram][105]: nvram crc check failed (0x... != 0x... 0x...)
```

Symptoms: solid red power LED that never turns green, LAN link flapping, no ARP
or IPv6 activity (the system crashes before networking starts). The factory
reset button has no effect. Recover via UART:

1. Interrupt autoboot: type `stop` during the 5-second countdown to reach
   `PRX300 #`.
2. Read NVRAM into RAM: `nand read 0x82000000 nvram 0x80000`.
3. Restore the original bytes with `mw.b` at the changed offsets
   (base MAC at `0x5D`, FSAN suffix at `0x69`), verifying with `md.b`.
4. Write back: `nand erase.part nvram` then `nand write 0x82000000 nvram 0x80000`.
5. `reset`.

After the device boots, make the change properly through the sysfs interface so
`libpalbsp.so` recomputes the CRC.

U-Boot reference:

```
Platform: PRX300 (Intel/Lantiq prx300, interAptiv)
U-Boot:   2016.07-CALIX-v3.1.311 (Sep 23 2024)
DRAM:     255 MiB
NAND:     256 MiB (Winbond 0xef 0xaa)
Boot:     5 s delay, type "stop" to interrupt
ipaddr:   192.168.1.1   serverip: 192.168.1.2 (tftpboot supported)
```

# Known Bugs

- **IPv6 firewall bypass (root).** `ip6tables` is never loaded; dropbear on
  port 30007 and telnet on 23 are reachable over IPv6 link-local with hardcoded
  `support` / `support`. See [Root procedure](#root-procedure).
- **LAN management dies on fiber connect.** EXOS tears down LAN-side management
  when the PON link comes up and does not restore it until reboot.
- **NVRAM CRC brick.** Raw writes to `mtd10` that skip the CRC recompute fail at
  boot and need UART recovery. Use the sysfs interface instead.

# Miscellaneous Links

- [Calix GigaPoint ONTs](https://www.calix.com/platforms/premises-systems.html)
- [Calix 100-05609 GPON ONT SFP](/ont-calix-100-05609) — the SFP sibling that
  runs the same EXOS stack inside the GigaSpire u6x

# Teardown and other photos

<ImageFigure file="ont-calix-gp1100g_teardown.png" alt="Calix GP1100G board" caption="Calix GP1100G: board with UART leads attached. MaxLinear PRX300 SoC (center), SC/APC fiber pigtail (white loop), NAND flash, and the 4-pin UART header on the right edge next to the RJ45 cage." />
