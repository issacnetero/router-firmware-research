# TP-Link Archer C6 v4.8 (IN) Hardware Research Notes

## Executive Summary

This document summarizes the hardware reconnaissance and research findings for the TP-Link Archer C6 v4.8 (India variant) in the context of potential OpenWrt support. The device uses a **TP1900BN/MT7626B** SoC, which currently lacks mainline Linux or OpenWrt kernel support. However, UART access has been successfully established, and the device has been confirmed to contain **8MB NOR flash** (EN25QH64), sufficient for OpenWrt deployment. This is a critical correction to the previously assumed 4MB limitation.

## Hardware Identification

| Component | Part Number | Details |
|-----------|-------------|---------|
| **SoC** | TP-Link TP1900BN | Rebranded MediaTek MT7626B, ARM Cortex-A7 |
| **Flash** | CFeon X40LW14 / EN25QH64 | 8MB NOR flash (64Mbit) |
| **RAM** | - | 32MB |
| **Network Switch** | Realtek RTL8397S | (Note: Boot log shows RTL8367D driver) |
| **Board Revision** | Archer C6, IN/4.8 | India variant |
| **U-Boot Version** | 2014.04-rc1 | Build date: Mar 31 2022 |

**Important Correction:** The flash chip is **EN25QH64**, which is 8MB (64Mbit), not 4MB as previously assumed by the OpenWrt community. This makes OpenWrt support technically feasible.

## Methodology

### UART Access

UART was successfully accessed using an ESP8266MOD as a USB-to-UART bridge via SoftwareSerial. The following configuration was used:

**ESP8266 Sketch (Passthrough Bridge):**
```cpp
#include <SoftwareSerial.h>

SoftwareSerial routerSerial(4, 5); // RX=GPIO4, TX=GPIO5

void setup() {
  Serial.begin(230400);
  routerSerial.begin(115200);
  Serial.println("Bridge ready...");
}

void loop() {
  if (routerSerial.available()) {
    Serial.write(routerSerial.read());
  }
  if (Serial.available()) {
    routerSerial.write(Serial.read());
  }
}
```

**UART Pinout:**
| Pad | Function | Notes |
|-----|----------|-------|
| TX | Router Transmit | Connect to ESP8266 RX (GPIO4) |
| RX | Router Receive | Connect to ESP8266 TX (GPIO5) |
| GND | Ground | Use TP2_GND_2 or USB shield ground |

**Serial Settings:**
| Parameter | Value |
|-----------|-------|
| Baud Rate | 115200 (router side) / 230400 (PC side via ESP8266) |
| Data Bits | 8 |
| Stop Bits | 1 |
| Parity | None |
| Flow Control | None |

### Boot Log Extraction

A complete boot log was captured successfully. Key findings:

**Flash Detection:**
```
EN25QH64(0x1c 0x70 0x17)
pagesize=0x100
sectsize=0x1000
chipsize=0x800000   // 8MB
readcap=0x3f
```

**U-Boot Environment Warning:**
```
NOR: *** Warning - bad CRC, using default environment
```
This indicates the U-Boot environment is corrupted or empty, likely resulting in `bootdelay=0`, preventing manual U-Boot interruption.

**Boot Process:**
```
U-Boot 2014.04-rc1 (Mar 31 2022 - 10:38:04)
[mt7626] - SoC confirmation throughout boot log
Register RTL8367D Ethernet Driver
GMAC1_MAC_ADRH -- : 0x00000019
GMAC1_MAC_ADRL -- : 0x66cb8b07
```

## Findings by Component

### SoC: MT7626 / TP1900BN

- ARM Cortex-A7 architecture
- Boot log confirms "Hello, MT7626 E2" message
- DBDC Mode=1 (dual-band concurrent)
- WiFi firmware loads from `/fw/mtk/` directory
- Existing reverse engineering reference: [GitHub - hacefresko/TP-Link-Archer-C6-EU-re](https://github.com/hacefresko/TP-Link-Archer-C6-EU-re)

### Flash: EN25QH64 (8MB)

- Sufficient size for OpenWrt (minimum 4MB required)
- NOR flash type
- Partitions visible in boot log via `mtdparts` (if accessible)

### WiFi

- Internal MediaTek WiFi
- Blobs located at `/fw/mtk/`:
  - `test_mcu.bin` (MCU firmware)
  - `test_wifi.bin` (WiFi firmware)
  - `WIFI_RAM_CODE_iemi.bin` (RAM code)
  - `WIFI_RAM_CODE_demi.bin`
  - `WIFI_RAM_CODE_7626.bin`
  - `mt7626_patch_e1_hdr.bin`

### Network

- RTL8367D Ethernet switch driver loaded
- MAC addresses visible:
  - GMAC1 MAC: `19:66:cb:8b:07`
- VLAN table configured for 5 ports (eth0-eth4)

### Serial Console

- Password prompt present at boot
- Unknown password (attempted: web admin password, admin, root, tplink, tplink1234)
- 5-attempt limit before lockout

## Current Roadblocks

| Issue | Status | Impact |
|-------|--------|--------|
| U-Boot shell access | Blocked | `bootdelay=0` prevents interrupt |
| Serial console login | Unknown password | Shell access unavailable |
| Mainline kernel support | Nonexistent | MT7626 not supported in Linux/OpenWrt |
| OpenWrt target | Absent | No mediatek/mt7626 target exists |
| WiFi driver | Proprietary blob | No open-source mt76 driver for MT7626 |

## Next Steps

### Option 1: Community Contribution (Recommended)

1. **Publish findings** on the OpenWrt Forum (Hardware Questions section) with:
   - Full UART boot log
   - Hardware identification details
   - Flash size confirmation (8MB)
   - PCB photos and UART pinout

2. **Collaborate** with OpenWrt developers to:
   - Establish MT7626 kernel support
   - Create device tree source (DTS) file
   - Integrate WiFi blobs

### Option 2: Independent Development Path

1. **CH341A flash dump** (₹300-400 investment):
   - Dump complete 8MB flash via SOIC8 clip
   - Extract partition table, U-Boot, kernel, rootfs
   - Extract WiFi calibration data (EEPROM/ART)

2. **Build OpenWrt port**:
   - Create `target/linux/mediatek/mt7626/` directory
   - Write board-specific DTS file
   - Port MT7622 base config to MT7626
   - Test via TFTP boot from U-Boot

### Option 3: SSH Exploit Path

The boot log shows OpenSSH 6.6.0 running on port 22. Attempt SSH access:
```bash
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 \
    -oHostKeyAlgorithms=+ssh-dss \
    -c aes256-cbc \
    admin@192.168.0.1
```

Potential passwords to try:
- Web admin password (user-set)
- `admin`
- `tplink`
- `1234`
- `root`
- (blank)

If successful, the following commands will provide critical data:
```bash
cat /proc/mtd
cat /proc/cpuinfo
dd if=/dev/mtd0 of=/tmp/flash.bin  # Requires writable /tmp
```

## Known Hardware Similarities

| Device | SoC | Notes |
|--------|-----|-------|
| Archer C6 EU v4 | MT7626 | Same PCB layout, same flash size confirmed |
| Archer C6 IN v4.8 | MT7626 | Same SoC, same 8MB flash |
| Archer C8 US | TP1900BN | Same SoC, may have compatible WiFi blobs |
| Archer A6 v4 | MT7626 | Under active research for OpenWrt |

## References

1. OpenWrt Forum: [TP-Link Archer c6 IN v4.8 thread](https://forum.openwrt.org/)
2. Reverse engineering: [GitHub - TP-Link-Archer-C6-EU-re](https://github.com/hacefresko/TP-Link-Archer-C6-EU-re)
3. OpenWrt Wiki: [Archer C6 v3](https://openwrt.org/toh/tp-link/archer_c6)
4. MT7626/MIPS: [mediatek/mt7626 kernel work](https://github.com/openwrt/openwrt/tree/main/target/linux/mediatek)

## Conclusion

The Archer C6 v4.8 (IN) contains 8MB flash and an MT7626B SoC. While OpenWrt support does not exist yet, the hardware is capable. This research provides the foundation for a potential port, with UART access established and critical hardware data captured. The next step should be either a community forum post to share findings or further access via SSH/CH341A flash dump.

---

**Research Conducted:** June 2026  
**Contributor:** Hardware Research Team  
**Status:** Phase 1 Complete (Hardware Reconnaissance)
