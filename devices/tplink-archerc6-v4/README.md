# Hardware Analysis & U-Boot Data: TP-Link Archer C6 v4 (MediaTek MT7626 / Leopard)

## Overview

This document compiles hardware discoveries, U-Boot environment logs, and boot constraints for the **TP-Link Archer C6 v4** router. Unlike previous revisions of the Archer C6 (which utilized Qualcomm or MediaTek MIPS architectures), the **v4** introduces a heavily locked-down, proprietary MediaTek ARM platform.

### Target Hardware Specifications

* **SoC:** MediaTek MT7626 (Internal Code: `leopard`)
* **Board Configuration:** `leopard_evb`
* **Vendor:** MediaTek / TP-Link
* **CPU Architecture:** ARMv7 (`cpu=armv7`)
* **Storage Interface:** SPI-NOR Flash (`snor`)
* **Default Baud Rate:** 115200

---

## 1. Complete U-Boot Environment Variables

The following variables were pulled via serial UART at the `MT7626>` bootloader prompt using the `printenv` command.

```property
baudrate=115200
board=leopard_evb
board_name=leopard_evb
bootcmd=jmpaddr 0x30030080
bootdelay=1
bootfile=lede_uImage
cpu=armv7
ethact=mtk_eth
ethaddr=00:0C:E7:11:22:33
fileaddr=40200000
filesize=60011a
ipaddr=192.168.0.1
loadaddr=0x40200000
serverip=192.168.0.3
soc=leopard
vendor=mediatek

Environment size: 334/4092 bytes

```

### Key Technical Insights from Environment:

* **Memory-Mapped Flash Offset:** `bootcmd=jmpaddr 0x30030080` confirms that the SPI-NOR flash memory is physically mapped to the standard MediaTek base address of `0x30000000`.
* **Partition Split:** The bootloader utilizes the first `0x30000` (192 KB) bytes of the flash space. The functional factory/kernel firmware partition begins exactly at execution address `0x30030000`, with an offset of `0x80` (128 bytes) reserved for vendor image headers.
* **Preferred RAM Safe-Zone:** `loadaddr=0x40200000` is the default volatile memory layout address for streaming binaries via TFTP.

---

## 2. Flash Memory Controller Behavior (`snor`)

The built-in help texts for this U-Boot build are severely stripped (`CONFIG_SYS_LONGHELP` disabled) to conserve space. However, direct boundary faulting revealed the correct arguments for the SPI-NOR manipulation tool.

### Command Help Output

```bash
MT7626> help snor
snor - snor   - spi-nor flash command

MT7626> help erase
erase - erase FLASH memory

```

### Syntax Verification

By testing null execution limits, the driver confirmed it accepts standard positional hexadecimal values for offset and length configurations:

```bash
MT7626> snor erase 0 0
Nor flash erase len: 0x0 done.

```

* **Validated Tool Syntax:** * `snor erase [flash_offset_hex] [length_hex]`
* `snor write [ram_address_hex] [flash_offset_hex] [length_hex]`



---

## 3. Firmware Validation & Boot Logs

Attempts to bypass storage writing and boot a foreign/MIPS OpenWrt binary image directly from volatile RAM space (`0x40200000`) triggered hard vendor-validation checks embedded inside the bootloader logic.

### Boot Execution Stream

```bash
MT7626> bootm 0x40200000
boot from bootm 0x40200000
## Booting image at 40200000 ...

Not tp Image, abort.

```

### Critical Security Discoveries:

1. **The "Not tp Image" Guard:** The U-Boot bootloader actively inspects image headers before handing control over to the program counter. If the signature does not precisely match the proprietary TP-Link image structures, the bootloader explicitly halts execution (`abort.`).
2. **Protection Against Bricking:** This validation acts as a protection rail, preventing unaligned or foreign CPU instruction sets from causing a fatal execution loop on the ARMv7 processor.

---

## 4. Primary Roadblocks for OpenWrt Development

For developers looking to carry this research forward, these are the fundamental hurdles that need solving:

* **Architecture Disconnect:** Existing OpenWrt images for older Archer C6 targets (v2, v3) are compiled for **MIPS** processors. The Archer C6 v4 requires a completely distinct toolchain targeting **ARMv7** instructions.
* **Driver Restrictions:** The MediaTek MT7626 ("Leopard") SoC features proprietary, closed-source networking drivers. Open-source Linux support currently lacks implementation wrappers for the built-in Ethernet switch driver and wireless network interface cards (NICs) on this silicon chip.
* **Storage Footprint:** The device leverages a restricted partition layout optimized for tightly compressed vendor files. Fitting a modern OpenWrt kernel along with a LuCI web user interface requires code reductions below standard sizing thresholds.
