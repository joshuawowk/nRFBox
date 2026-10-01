# nRFBox — Device Assembly & Flashing Guide

This guide walks you through building an **nRFBox v3** and getting firmware onto it —
from an unpopulated PCB (or a bare ESP32 + modules on a breadboard) all the way to a
booting, menu-driven device.

It is written to match the hardware and firmware in **this** repository:

- Schematic: [`Schematic/nRFBOX-Schematic.jpg`](Schematic/nRFBOX-Schematic.jpg)
- PCB manufacturing files: [`PCB/`](PCB) (`Gerber.zip`, `BOM.xlsx`, `CPL.csv`, `PCB Print.jpg`)
- Firmware source: [`nRFBox/`](nRFBox)
- Pre-compiled binaries: [`Pre-compiled Bin/`](Pre-compiled%20Bin)
- Flashing helpers: [`Flash File/`](Flash%20File) (`flash_download_tool_3.9.5.zip`, `nRFBOX.partitions.bin`)

---

## ⚠️ Legal & Safety Notice — read first

The nRFBox includes **jamming** (2.4 GHz constant-carrier), **BLE jamming/spoofing**, and
**Wi‑Fi deauthentication** features. These transmit on purpose to disrupt other devices.

- **Operating a jammer or deauther against networks/devices you do not own or have written
  permission to test is illegal in most countries** (e.g. FCC Part 15 in the US, the Wireless
  Telegraphy Act / Ofcom in the UK, and equivalents elsewhere). Penalties can include heavy
  fines and equipment seizure.
- Use these features **only** in an RF-shielded environment or against your own hardware, for
  education, research, and authorized security testing.
- The scanner, analyzer, BLE scan, and Wi‑Fi scan features are **passive/receive-only** and are
  generally fine to use for learning.

**Battery safety:** this board charges a single Li‑ion cell via a TP4056. Use a protected cell,
never charge unattended, observe polarity, and don't short the pack.

You are responsible for how you use this device.

---

## 1. What you are building

nRFBox v3 is a self-contained 2.4 GHz toolkit on a single PCB:

| Block | Part | Role |
|-------|------|------|
| MCU | **ESP32‑WROOM‑32U** (U1) | Runs the firmware; native Wi‑Fi + BLE radio |
| Radios | **3 × nRF24L01** (U5, U6, U7) | 2.4 GHz scan / analyze / jam (shared SPI bus) |
| Display | **SSD1306 0.96" OLED** (U4) | 128×64 menu UI over I²C |
| RGB LED | **WS2812B** (D1) | Status indicator |
| Charger | **TP4056** (IC1) | Single-cell Li‑ion charging over USB‑C |
| Regulator | **LF33** (U3) | 3.3 V rail |
| USB‑UART | **CP2102** (U2) | Programming + serial, with auto-reset (Q1/Q2) |
| Storage | **micro‑SD** (U8) | On-device firmware update |
| Antennas | 3 × U.FL / SMA (ANT1‑3 / J2‑J4) | External 2.4 GHz antennas |
| Input | 5 × navigation buttons + RESET + BOOT | UI + flashing |

You can build this **three** ways:

- **Path A — Order it assembled** (recommended for most people). Send the Gerbers + BOM + CPL
  to a PCBA house (e.g. JLCPCB) and let them place the SMD parts. You solder only the through-hole
  / mechanical bits.
- **Path B — Hand-assemble the PCB.** Reflow/hand-solder all SMD parts yourself.
- **Path C — Breadboard / dev-board build.** Wire a plain ESP32 dev board to modules using the
  pin map in [§4](#4-pin-map-single-source-of-truth). No custom PCB required — great for trying
  the firmware before committing to a board.

---

## 2. Bill of Materials

Full quantities are in [`PCB/BOM.xlsx`](PCB/BOM.xlsx). Summary:

### Core / active parts
| Qty | Part | Designator | Notes |
|----:|------|------------|-------|
| 1 | ESP32‑WROOM‑32U | U1 | **‑32U** = external antenna (U.FL). |
| 3 | nRF24L01 (GT24 mini SMD) | U5, U6, U7 | Running all three at once is power-hungry — see notes. |
| 1 | SSD1306 0.96" OLED, white | U4 | I²C, addr `0x3C`. |
| 1 | WS2812B RGB LED (5050) | D1 | |
| 1 | TP4056 | IC1 | Li‑ion charger. |
| 1 | LF33 (3.3 V LDO, D‑PAK) | U3 | |
| 1 | CP2102 (QFN‑28) | U2 | USB‑to‑UART. |
| 2 | S8050 NPN (SOT‑23) | Q1, Q2 | Auto-reset for flashing. |
| 1 | micro‑SD holder | U8 | |

### Connectors / mechanical
| Qty | Part | Designator |
|----:|------|------------|
| 1 | USB‑C receptacle (TYPE‑C‑31‑M‑12) | J1 |
| 3 | SMA jack (edge) | J2, J3, J4 |
| 3 | U.FL connector | ANT1, ANT2, ANT3 |
| 1 | SPDT slide switch | SW1 |
| 2 | Push button, SMD (RESET, BOOT) | B1, B2 |
| 5 | Push button, 4‑pin SMD (navigation) | B3–B7 |
| 1 | Li‑ion cell / holder | BAT |

### Passives
| Qty | Value | Package |
|----:|-------|---------|
| 9 | 100 nF | 0805 |
| 4 | 10 µF | 0805 |
| 2 | 10 µF | 1206 |
| 5 | Tantalum (bulk) | 1206 |
| 2 | 2.2 µF | 0805 |
| 2 | 1 µF | 0805 |
| 2 | 4.7 k | 0805 (R1, R3 — EN/GPIO0 pull-ups) |
| 5 | 1 k | 0805 |
| 1 | 1 k | 0805 (R14 — WS2812 data series) |
| 4 | 10 k | 0805 (R16–R19 — SD lines) |
| 1 | 10 k | 0805 (R13) |
| 2 | 12 k | 0805 (R4, R5 — auto-reset) |
| 2 | 5.1 k | 0805 (R7, R8 — USB‑C CC) |
| 1 | 470 Ω | 0805 (R2) |
| 1 | 390 k | 0805 (R15 — OLED IREF) |
| 4 | LED | 0805 (CHG, STD, STD1, STD2) |

### You also need (not on the BOM)
- 3 × 2.4 GHz antennas (U.FL pigtail → SMA, or SMA whip antennas)
- 1 × Li‑ion cell (e.g. 3.7 V pouch or 18650 with tabs)
- USB‑C cable (data-capable, not charge-only)
- Enclosure / 3D-printed case (optional)

---

## 3. Tools & materials

**Path A (assembled board):** fine-tip soldering iron, flux, solder, tweezers, isopropyl
alcohol, multimeter.

**Path B (hand SMD):** all of the above **plus** hot-air rework station or reflow hotplate,
solder paste, stencil (order with the PCB), and magnification. The CP2102 (QFN‑28) and USB‑C
connector are the hardest parts — do those first while the board is empty.

**Path C (breadboard):** ESP32 dev board, 1–3 nRF24L01 modules (+ 10 µF caps across each
module's VCC/GND!), an SSD1306 I²C OLED, jumper wires, breadboard.

---

## 4. Pin map (single source of truth)

These are defined in [`nRFBox/config.h`](nRFBox/config.h) and confirmed against the schematic.
**If you wire your own board, match this exactly** — the firmware has the pins hard-coded.

### Navigation buttons — `INPUT_PULLUP`, active LOW (button to GND)
| Function | GPIO | Board ref |
|----------|-----:|-----------|
| LEFT  | 25 | B3 |
| UP    | 26 | B5 |
| RIGHT | 27 | B4 |
| DOWN  | 32 | B6 |
| SELECT | 33 | B7 |

Plus **RESET** (chip EN) and **BOOT** (GPIO0) buttons for flashing (B1, B2).

### OLED — I²C (hardware I²C)
| Signal | GPIO |
|--------|-----:|
| SDA | 21 |
| SCL | 22 |

Default address `0x3C`. The firmware uses `U8G2_SSD1306_128X64_NONAME_F_HW_I2C`.

### nRF24L01 ×3 — shared VSPI bus
| Signal | GPIO |
|--------|-----:|
| SCK  | 18 |
| MISO | 19 |
| MOSI | 23 |

| Radio | Firmware name | CE | CSN | Board ref |
|-------|---------------|---:|----:|-----------|
| A | `RadioA` | 5  | 17 | U7 |
| B | `RadioB` | 16 | 4  | U5 |
| C | `RadioC` | 15 | 2  | U6 |

> On a breadboard you can start with **one** module wired as Radio A (CE=5, CSN=17). Scanner,
> Analyzer, and the BLE/Wi‑Fi tools work with a single nRF24; the jammer/proto-kill just get
> stronger with all three.

### WS2812 RGB status LED
| Signal | GPIO |
|--------|-----:|
| DIN | 14 (through R14, 1 k) |

### micro‑SD (firmware update only) — same SPI bus
| Signal | GPIO |
|--------|-----:|
| CS   | 5 (**shared with Radio A CE**) |
| SCK  | 18 |
| MOSI | 23 |
| MISO | 19 |

> **Design note — GPIO5 is shared** between the SD card CS and nRF24 Radio A's CE. This is fine
> because they are never used at the same time: the radios sit powered-down in a neutral state
> while you're in **Setting → Update Firmware**. Don't try to use the SD card while a radio mode
> is active.

> **Strapping pins:** GPIO2 and GPIO15 (Radio C) and GPIO5 are ESP32 strapping/boot pins. The
> nRF24 modules keep them in a safe state at reset, but if you see boot problems on a hand-wired
> build, disconnect the radios and retest.

---

## 5. Assembly

### Path A — Order the board assembled (JLCPCB example)
1. Upload [`PCB/Gerber.zip`](PCB/Gerber.zip) to the PCB order page. Confirm 2-layer, 1.6 mm.
2. Enable **SMT assembly**. Upload the BOM ([`PCB/BOM.xlsx`](PCB/BOM.xlsx)) and the placement
   file ([`PCB/CPL.csv`](PCB/CPL.csv)).
3. In the assembly preview, check each part's rotation and polarity — **LEDs, the WS2812
   (D1), TP4056 (IC1), tantalum caps, and the CP2102 (U2)** are the usual suspects for wrong
   rotation. Compare against [`PCB/PCB Print.jpg`](PCB/PCB%20Print.jpg).
4. The ESP32‑WROOM‑32U, nRF24 modules, SMA jacks, slide switch, and battery are commonly placed
   as **hand-solder / DNP** parts depending on the assembler — plan to solder those yourself.
5. When the board arrives, hand-solder any remaining through-hole/mechanical parts (SMA jacks,
   slide switch, battery leads).

### Path B — Hand assembly order
Work from smallest/most-heat-sensitive to largest:
1. **CP2102 (U2, QFN‑28)** and **USB‑C (J1)** first, while the board is clear. Verify no bridges
   under magnification.
2. Passives (resistors, then caps). Do the **tantalum caps and LEDs with correct polarity.**
3. **TP4056 (IC1)**, **LF33 (U3)**, transistors **Q1/Q2**.
4. **ESP32‑WROOM‑32U (U1)**.
5. **OLED (U4)**, **WS2812 (D1)**, **micro‑SD holder (U8)**.
6. **nRF24 modules (U5–U7)** — add a **10 µF** cap across each module's VCC/GND if not already on
   the footprint; this is the single biggest reliability fix for nRF24.
7. Mechanical: **SMA jacks (J2–J4)**, **slide switch (SW1)**, **buttons**, **battery**.

### Path C — Breadboard build
Wire per [§4](#4-pin-map-single-source-of-truth). Minimum viable rig = ESP32 dev board + SSD1306
OLED (SDA=21, SCL=22) + one nRF24 (CE=5, CSN=17, SCK=18, MISO=19, MOSI=23) with a 10 µF cap on
its supply. Add buttons to GPIO 25/26/27/32/33 (other side to GND). The WS2812 on GPIO14 is
optional.

### Antennas
Fit **all three** nRF24 antennas before transmitting — running an nRF24 output stage without an
antenna load can damage it. Connect U.FL pigtails to the SMA jacks (or fit SMA whips directly).

---

## 6. First power-on smoke test (before flashing)

1. **Do not** install the battery yet. Inspect for solder bridges, especially around U1, U2, and
   the USB‑C connector.
2. Plug in USB‑C. Measure the **3.3 V rail** at a convenient test point / OLED VCC — it should
   read ~3.30 V. If it's low/zero, stop and debug the LF33 / TP4056 before continuing.
3. The charge LED (CHG) may light. The board can be powered from USB with no battery.
4. Confirm the host PC enumerates a **CP2102 USB‑UART** (a new `COMx` on Windows or
   `/dev/ttyUSB0` on Linux/`/dev/tty.usbserial-*` on macOS). If not, install the
   **Silicon Labs CP210x VCP driver**.

If all three check out, you're ready to flash.

---

## 7. Flashing the firmware

You have three routes. **If your ESP32 has never run nRFBox (blank flash), use Method 1** — it
writes the bootloader, partition table, and app in one shot. Methods 2 and 3 are for
re-flashing/updating a board that already has the bootloader.

### Method 1 — Build & upload from source (recommended)

**1. Install the Arduino IDE** (2.x) and the ESP32 core:
- File → Preferences → *Additional Boards Manager URLs*:
  `https://espressif.github.io/arduino-esp32/package_esp32_index.json`
- Tools → Board → Boards Manager → install **esp32 by Espressif Systems**.
  *(This project’s pre-compiled binaries were built on the ESP32 Arduino core v2.x / IDF 4.4.5.
  The 2.0.x core line is the known-good target.)*

**2. Install libraries** (Tools → Manage Libraries):
- **U8g2** (by oliver)
- **RF24** (by TMRh20)
- **Adafruit NeoPixel**

Wi‑Fi, BLE, SD, SPI, Wire, EEPROM, Preferences, and Update come with the ESP32 core.

**3. Open the sketch:** open [`nRFBox/nRFBox.ino`](nRFBox/nRFBox.ino). Keep all the `.cpp`/`.h`
files in the same `nRFBox/` folder — they compile as part of the sketch.

**4. Board settings** (Tools menu):
| Setting | Value |
|---------|-------|
| Board | **NodeMCU‑32S** (or *ESP32 Dev Module*) |
| Upload Speed | 921600 (drop to 115200 if uploads fail) |
| Flash Frequency | 80 MHz |
| Flash Mode | QIO (DIO if it won't boot) |
| **Partition Scheme** | **Minimal SPIFFS (1.9 MB APP with OTA)** |
| Port | your CP2102 port |

> **Partition scheme matters.** The firmware is ~1.5 MB and the SD "Update Firmware" feature needs
> an **OTA-capable** layout. "Minimal SPIFFS (1.9MB APP with OTA)" satisfies both. The default
> partition table is too small and the build/OTA will fail.

**5. Upload.** The CP2102 auto-reset (Q1/Q2) should put the ESP32 into download mode
automatically. If you see `Connecting........_____`, **hold BOOT (B2)**, tap **RESET (B1)**,
release BOOT, and let it upload.

### Method 2 — Flash a pre-compiled binary with `esptool`

Binaries are in [`Pre-compiled Bin/`](Pre-compiled%20Bin) (use the highest version, currently
`nRFBox_V2-7-2.ino.node32s.bin`). Install esptool: `pip install esptool`.

The Arduino-exported app binary flashes at **`0x10000`**. A blank ESP32 also needs the stock
**bootloader** (`0x1000`) and **boot_app0** (`0xe000`) plus this project's **partition table**
(`0x8000`, [`Flash File/nRFBOX.partitions.bin`](Flash%20File/nRFBOX.partitions.bin)).

**Re-flashing a board that already ran nRFBox** (bootloader already present) — app + partitions
is enough:
```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 921600 write_flash -z \
  0x8000  "Flash File/nRFBOX.partitions.bin" \
  0x10000 "Pre-compiled Bin/nRFBox_V2-7-2.ino.node32s.bin"
```

**Blank chip** — you also need `bootloader.bin` and `boot_app0.bin` from the ESP32 Arduino core.
Find them under your core install, e.g.
`…/packages/esp32/hardware/esp32/<ver>/tools/partitions/boot_app0.bin` and the matching
`…/tools/sdk/esp32/bin/bootloader_qio_80m.bin`:
```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 921600 write_flash -z \
  0x1000  bootloader_qio_80m.bin \
  0x8000  "Flash File/nRFBOX.partitions.bin" \
  0xe000  boot_app0.bin \
  0x10000 "Pre-compiled Bin/nRFBox_V2-7-2.ino.node32s.bin"
```
> If unsure which files a blank chip needs, just use **Method 1** — it's less error-prone.

Windows users can instead use the GUI **Espressif Flash Download Tool** shipped in
[`Flash File/flash_download_tool_3.9.5.zip`](Flash%20File/flash_download_tool_3.9.5.zip):
choose chip **ESP32**, add the same files at the same offsets, select the COM port, set
40 MHz/DIO or 80 MHz/QIO, and press **START**.

### Method 3 — On-device update from micro‑SD (no PC)
Once a board is already running nRFBox:
1. Copy a pre-compiled binary to the **root of a FAT32 micro‑SD card** and rename it exactly
   **`firmware.bin`**.
2. Insert the card, power on, and from the main menu open **Setting → Update Firmware**, then
   press SELECT.
3. The device reads the file over SPI and re-flashes itself (OTA), then reboots. If it reports
   `SD Init Failed` or `File Not Found`, re-check the card format and filename.

---

## 8. First boot & controls

On boot you'll see the CiferTech splash, then a 6‑icon grid menu.

| Button | In the menu | Inside a tool |
|--------|-------------|---------------|
| LEFT / RIGHT | Move selection | Change value / option (tool-dependent) |
| UP / DOWN | Move selection by a row | Navigate sub-options |
| SELECT | Enter the highlighted tool | **Press again to return to the menu** |

Menu items: **Scanner, Analyzer, WLAN Jammer, Proto Kill, BLE Jammer, BLE Spoofer, Sour Apple,
BLE Scan, WiFi Scan, Deauther, About, Setting**.

The WS2812 LED reflects activity (e.g. purple during scanning, red while jamming, orange while a
deauth attack is running). You can toggle the LED and OLED brightness in **Setting**.

---

## 9. Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| **OLED blank** | I²C wiring (SDA=21, SCL=22), no 3.3 V, or wrong panel. The firmware assumes addr `0x3C`. Check the rail from [§6](#6-first-power-on-smoke-test-before-flashing). |
| **No serial port** | Install the **CP210x VCP driver**; try a data-capable USB‑C cable. |
| **Upload stuck at `Connecting…`** | Hold **BOOT (B2)** during connect; lower upload speed to 115200; check auto-reset transistors Q1/Q2 & R4/R5. |
| **Boots, but radios show "Inactive"** | nRF24 power/decoupling. Add a **10 µF** cap on each module's supply. Running **all three** nRF24s can brown out a weak supply — test with one first. |
| **Device resets / freezes when jamming** | Same power issue: three radios at PA_MAX draw a lot. Use a healthy battery, charge it, or reduce PA level in the Jammer menu. |
| **`SD Init Failed` on firmware update** | FAT32 card, file named exactly `firmware.bin` at the card root; remember SD CS = GPIO5, so exit any radio tool first. |
| **Build fails: "sketch too big" / OTA errors** | Wrong partition scheme — select **Minimal SPIFFS (1.9MB APP with OTA)**. |
| **Weak range / nothing happens on TX tools** | Antennas not fitted, or you're at range. Never transmit with an nRF24 that has no antenna. |

---

## 10. Reference: firmware structure

| File | Responsibility |
|------|----------------|
| [`nRFBox.ino`](nRFBox/nRFBox.ino) | Main menu, button handling, dispatch to each tool's setup/loop |
| [`config.h`](nRFBox/config.h) | **Pin definitions**, includes, per-tool namespaces |
| [`setting.cpp`](nRFBox/setting.cpp) / [`setting.h`](nRFBox/setting.h) | Radio init, EEPROM settings, OLED text helpers, firmware-update UI |
| [`ism.cpp`](nRFBox/ism.cpp) | Scanner, Analyzer, Jammer, Proto Kill (nRF24) |
| [`bluetooth.cpp`](nRFBox/bluetooth.cpp) | BLE Jammer, BLE Scan, Sour Apple, BLE Spoofer |
| [`wifi.cpp`](nRFBox/wifi.cpp) | Wi‑Fi Scan, Deauther |
| [`neopixel.cpp`](nRFBox/neopixel.cpp) | WS2812 status colors |
| [`icon.h`](nRFBox/icon.h) | XBM bitmaps and UI text |

---

*For the full feature walkthrough, see the upstream
[nRFBox Wiki](https://github.com/cifertech/nRFBox/wiki). This build/flash guide is maintained
alongside the source in this repository.*
