# TP-Link Archer C6 v4.8 (IN) — SSH Access & Forum Draft Notes

## 1. SSH Connection Attempts

### 1.1 Initial Discovery

The boot log revealed that **OpenSSH 6.6.0** is running on **port 22**. The SSH banner displayed:

```
TPOS 5 IPSSH Test
```

This confirms the SSH daemon is active but uses **legacy encryption algorithms** due to its older version.

### 1.2 Connection Commands

The following commands were attempted from **WSL Ubuntu** (Windows Subsystem for Linux) on the same local network via Ethernet (LAN port):

```bash
# Basic connection attempt (failed due to key exchange mismatch)
ssh admin@192.168.0.1

# Full legacy-compatible command
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 \
    -oHostKeyAlgorithms=+ssh-dss \
    -oMACs=+hmac-sha1 \
    -c aes256-cbc \
    admin@192.168.0.1
```

### 1.3 Error Responses

| Attempt | Result |
|---------|--------|
| Default SSH command | `Unable to negotiate: no matching key exchange method found` |
| With legacy flags | Connection established, but `Permission denied (password)` |
| After adding MACs flag | Connection closed by remote host (wrong password) |

### 1.4 Usernames Tested

| Username | Password Attempts | Result |
|----------|-------------------|--------|
| `admin` | Web admin password, Secureme#2bit, 17898911, various hashes | Permission denied |
| `root` | Same as above | Permission denied |
| `tplink` | tplink, tplink123, secureme | Permission denied |

### 1.5 Password Hash Generation Attempts

Using the router's LAN MAC address (`8C:86:DD:27:55:72`), MD5 hashes were generated:

```bash
echo -n "8C:86:DD:27:55:72admin" | md5sum
# Output: 732477cdcb46fc7d468ff556b4b114b7

echo -n "8c:86:dd:27:55:72admin" | md5sum
# Output: 44ad8f73b1edfc63393b1fb0cc9ec9a8

echo -n "8C-86-DD-27-55-72admin" | md5sum
# Output: f3d291bf8f6168cc444afa2a77396cc5

echo -n "8c-86-dd-27-55-72admin" | md5sum
# Output: ce05e6c0f44790e9efe45faa16cac29b

echo -n "8C:86:DD:27:55:72root" | md5sum
# Output: d46956c97d1bc26de648f5202c19463c

echo -n "8c:86:dd:27:55:72root" | md5sum
# Output: aa81a30c8ad5ecb94a6fa5a6fbe40e95
```

**None of these hashes worked** as SSH passwords.

### 1.6 Firmware Password Extraction Attempt

The official firmware file (`Archer-C6-4.0-TPOS-up-noboot-128B_2025-07-30_10.59.50.bin`) was extracted using `binwalk`, but the TPOS firmware uses a **proprietary compressed format** (LZMA chunks) that cannot be mounted or read as a standard filesystem. No `/etc/passwd` or `/etc/shadow` files were found.

**Strings extraction** from the raw binary revealed:

```
root-servers
PPPOE4_password:
password length is illegal (1-32) or blank space!
SSID1=TPLINK_5G_935E
SSID1=TPLINK_935E
PASS dummypwd@
web/modules/advanced/system/admin
web/modules/advanced/system/admin/changePassword
```

No usable password was found.

### 1.7 Conclusion on SSH

The TPOS SSH implementation is **locked to TP-Link's proprietary Tether app** and does not accept human-generated passwords via standard authentication. The password is likely **hardcoded in the binary** but stored in a custom format that cannot be extracted without deeper reverse engineering.

---

## 2. OpenWrt Forum Draft Post

> **Title:** TP-Link Archer C6 IN v4.8 — UART Boot Log & Hardware Research
>
> **Category:** Hardware Questions
>
> ---
>
> **Hello everyone,**
>
> I have been researching the **TP-Link Archer C6 IN v4.8** (India variant) for OpenWrt support. Below is a summary of my hardware reconnaissance, UART access, and findings. I hope this contributes to a potential port.
>
> ---
>
> ### Hardware Identification
>
> | Component | Part Number / Details |
> |-----------|-----------------------|
> | **SoC** | TP-Link TP1900BN (rebranded MediaTek MT7626B), ARM Cortex-A7 |
> | **Flash** | EN25QH64 — **8MB NOR** (64Mbit) |
> | **RAM** | 32MB |
> | **Switch** | RTL8367D (driver loads as RTL8367D) |
> | **Board Revision** | Archer C6, IN/4.8 |
> | **U-Boot Version** | 2014.04-rc1 (Mar 31 2022) |
> | **HVIN (Internal)** | Archer A6V4 |
>
> **Important Correction:** The flash chip is **EN25QH64 = 8MB**, not 4MB as previously assumed. This makes OpenWrt support technically feasible.
>
> ---
>
> ### UART Access
>
> **Location:** Component side, near the main SoC.
>
> **Pinout:**
> | Pad | Function |
> |-----|----------|
> | TX | Transmit |
> | RX | Receive |
> | GND | Ground (TP2_GND_2) |
>
> **Serial Settings:**
> - Baud: 115200 (router side)
> - Data Bits: 8
> - Stop Bits: 1
> - Parity: None
> - Flow Control: None
>
> **Access Method:** Used an ESP8266MOD as a USB-to-UART bridge via SoftwareSerial (GPIO4/GPIO5).
>
> ---
>
> ### Boot Log Findings
>
> **Flash Detection:**
> ```
> EN25QH64(0x1c 0x70 0x17)
> pagesize=0x100
> sectsize=0x1000
> chipsize=0x800000   // 8MB
> readcap=0x3f
> ```
>
> **U-Boot:**
> ```
> U-Boot 2014.04-rc1 (Mar 31 2022 - 10:38:04)
> NOR: *** Warning - bad CRC, using default environment
> ```
> U-Boot environment appears to be **corrupted or empty**, likely resulting in `bootdelay=0`, which prevents manual interruption.
>
> **SoC Confirmation:**
> ```
> Hello, MT7626 E2
> [mt7626] — repeated throughout the boot log
> ```
>
> **Ethernet:**
> ```
> Register RTL8367D Ethernet Driver
> GMAC1_MAC_ADRH -- : 0x00000019
> GMAC1_MAC_ADRL -- : 0x66cb8b07
> ```
>
> **WiFi:**
> ```
> [mt7626] <<<<<<<<<<<<< oooooo! will load file: /fw/mtk/WIFI_RAM_CODE_iemi.bin
> [mt7626] <<<<<<<<<<<<< oooooo! will load file: /fw/mtk/WIFI_RAM_CODE_demi.bin
> [mt7626] <<<<<<<<<<<<< oooooo! will load file: /fw/mtk/WIFI_RAM_CODE_7626.bin
> ```
> WiFi blobs are stored in `/fw/mtk/` inside flash.
>
> **Serial Console:**
> ```
> Password:
> ```
> Serial console is password-protected. Unknown password.
>
> ---
>
> ### SSH Discovery
>
> The boot log shows:
> ```
> OpenSSH 6.6.0 on port 22
> ```
>
> SSH is active but uses **legacy algorithms**:
> - Kex: `diffie-hellman-group1-sha1`, `diffie-hellman-group14-sha1`
> - MACs: `hmac-sha1`, `hmac-sha1-96`, `hmac-md5`, `hmac-md5-96`
> - Cipher: `aes256-cbc`
>
> **Connection Attempts:**
> ```bash
> ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 \
>     -oHostKeyAlgorithms=+ssh-dss \
>     -oMACs=+hmac-sha1 \
>     -c aes256-cbc \
>     admin@192.168.0.1
> ```
>
> **Result:** Connection succeeds but authentication fails. The SSH password appears to be **hardcoded in the TPOS firmware** and not retrievable via standard firmware extraction methods.
>
> ---
>
> ### Related Devices
>
> - **Archer C6 EU v4** — Same SoC and PCB layout, 8MB flash confirmed.
> - **Archer A6 v4** — Same internal hardware (HVIN shows Archer A6V4).
> - **Archer C8 US** — Same TP1900BN SoC.
>
> **Reference:** [GitHub — hacefresko/TP-Link-Archer-C6-EU-re](https://github.com/hacefresko/TP-Link-Archer-C6-EU-re)
>
> ---
>
> ### What's Still Needed for OpenWrt
>
> | Component | Status |
> |-----------|--------|
> | MT7626 kernel support | ❌ Not in mainline Linux or OpenWrt |
> | Device Tree Source (DTS) | ❌ Not written yet |
> | U-Boot environment access | ❌ Blocked by `bootdelay=0` |
> | WiFi driver | ❌ Proprietary `mt_wifi` blob only |
> | Flash partition map | ❌ Not yet extracted |
>
> ---
>
> ### Next Steps
>
> 1. **CH341A flash dump** — To extract partition layout, U-Boot environment, and WiFi calibration data.
> 2. **Kernel porting** — Establish MT7626 support in OpenWrt's `mediatek` target.
> 3. **DTS creation** — Write board-specific device tree for Archer C6 v4.8.
>
> I am happy to provide additional data, UART logs, or PCB photos. If anyone with kernel experience is interested in picking this up, please reach out.
>
> ---
>
> **Device is open and UART is connected.** Full boot log available upon request.

---

## 3. Summary

| Task | Status |
|------|--------|
| Hardware identification | ✅ Complete |
| UART access | ✅ Working (115200 baud) |
| Boot log capture | ✅ Complete |
| Flash size confirmation | ✅ 8MB (EN25QH64) |
| SSH access | ❌ Blocked (unknown hardcoded password) |
| Serial console login | ❌ Blocked |
| U-Boot shell | ❌ Blocked (`bootdelay=0`) |
| Firmware password extraction | ❌ TPOS proprietary format |
| OpenWrt forum post draft | ✅ Ready |
| CH341A flash dump | ⏳ Pending |
| OpenWrt port | ⏳ Not started |
