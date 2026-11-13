# Universal USB Flasher V2: design plan

> **Superseded.** This folder is the early V2 plan (CH343G). The current design is **V2.1** (CH342F dual UART, ESP-Prog port, current-limited target power, header ESD) in **[ElectroIoT/Universal-USB-Flasher-V2](https://github.com/ElectroIoT/Universal-USB-Flasher-V2)**. Use that repo for the BOM, pin list and EasyEDA checklists.


Status: **plan + netlist + LCSC BOM only.** Schematic capture and PCB layout are done by hand in EasyEDA.
Nothing here has been built or tested on hardware yet.

V2 keeps the V1 idea: one small USB-C board that flashes ESP32, ESP8266 (ESP-12F, ESP-01) and Arduino/AVR chips with no button pressing.
It adds true 3.3 V / 5 V logic levels, a plug-in ESP-01 socket, bare-module pull-ups, an Arduino reset pin, UPDI mode and USB overcurrent protection.

---

## 1. What was wrong or missing in V1

| # | V1 issue | V2 fix |
|---|----------|--------|
| 1 | **Q2 (IO0 transistor) is drawn with collector on DTR and emitter on IO0**, which is backwards. In that "reverse-active" mode the NPN has very low gain, so auto-boot is unreliable. Q1 (EN) is correct. | Q2: **C = IO0, B = RTS# via 10k, E = DTR#** (standard NodeMCU circuit). |
| 2 | H2/H3 "SJ_DTR / SJ_RTS" are 2-pin headers (PZ254V-11-02P), not solder jumpers. Without jumper caps auto-program does nothing, but the README says "closed by default". | SJ1/SJ2 are real **solder jumpers, bridged by default** (cut the trace to disable). |
| 3 | Pin 6 silkscreen says "RST" but it is raw RTS. There is no real reset pin for Arduino. | Dedicated **RST pin = DTR# through 100 nF** (classic Arduino auto-reset) plus a 10k pull-up. |
| 4 | CH340C runs at 3.3 V, so a 5 V Arduino's TX drives 5 V into a 3.3 V input. | **CH343G with VIO pin**: a slide switch sets the UART logic to 3.3 V or 5 V. |
| 5 | No fuse: a short on the 5V pin goes straight back to the PC USB port. | **500 mA polyfuse** (1 A trip) on VBUS. |
| 6 | Power LED 220 Ω from 5 V (~13 mA, very bright). | 1 kΩ from 3.3 V (~1.3 mA). |
| 7 | "PWD" silkscreen typo; README resistor values do not match the BOM (R1/R2 are 5.1k CC resistors, R3 is 220 Ω, R6/R7 are 1k). | Corrected in V2 docs and silkscreen. |
| 8 | The 100k base pull-downs (R8/R9) are not needed. | Removed. This also keeps the CH343 DTR pin free of a pull-down (see 3.2). |
| 9 | Three discrete ESD diodes. | One **USBLC6-2SC6** (D+, D- and VBUS). |

---

## 2. Feature list (V2)

1. USB-C (5.1k CC pull-downs, works with C-to-C cables).
2. **CH343G** USB-UART (SOP-16, hand-solderable). Up to 6 Mbaud. Native CDC driver on Windows 10/11, macOS and Linux (the WCH driver is optional).
3. **3.3 V / 5 V logic switch (SW3)**: sets CH343 VIO, so the TX, RX, DTR and RTS levels match the target.
4. **Auto-program for ESP32/ESP8266** (2× MMBT3904, NodeMCU-style), with **SJ1/SJ2** solder jumpers to disable it.
5. **Arduino auto-reset pin (RST)**: DTR# → 100 nF → RST, plus a 10k pull-up to VIO. Works with bare ATmega328P, Pro Mini, etc.
6. **UPDI mode (SW4)**: connects TX to RX through 4.7k, so the RX pin becomes a SerialUPDI pin. Flashes ATtiny 0/1/2-series and AVR-DA/DB/DD with megaTinyCore/DxCore ("SerialUPDI" programmer) or `pymcuprog`.
7. **ESP-01 / ESP-01S socket (J2, 2×4)**: plug the module in and flash, with no wires. GPIO2 and RST pull-ups are on the board.
8. **Bare ESP-12F / ESP32-WROOM support**: EN and IO0 10k pull-ups, EN 100 nF, and an **IO15 pin with a 10k pull-down**. A bare ESP-12F (or a pogo-pin jig) needs only the header wires.
9. RESET (EN) and BOOT (IO0) buttons.
10. LEDs: PWR (red), USB ACT (green, from CH343 ACT#), TX (orange), RX (yellow).
11. 500 mA polyfuse + Schottky back-feed diode + 800 mA 3.3 V LDO with 22 µF bulk at the regulator and at the ESP-01 socket.
12. Colour-coded silkscreen, kept from V1.

Non-goals for V2: SWD/JTAG, an AVR ISP (SPI) programmer, a target power on/off switch, and a 5V-mode warning LED (no DPDT slide switch was in stock at LCSC on 2026-11-12; see 6).

---

## 3. Circuit description

### 3.1 Power

```
USB-C VBUS ─┬─ U3 USBLC6 pin5 (VBUS clamp)
            └─ F1 500mA PTC ── VBUS_F ─┬─ U2 CH343G pin16 VDD5, pin7 VBUS
                                       ├─ C1 10µF
                                       └─ D1 1N5819HW ── +5V ─┬─ U1 SGM2212-3.3 VIN (C5 10µF)
                                                              ├─ H1 pin2 (5V out, ~4.6 V)
                                                              └─ SW3 throw "5V"
U1 VOUT ── +3V3 ─┬─ C6 22µF, C7 100nF
                 ├─ H1 pin1, J2 VCC (C8 22µF at socket)
                 ├─ pull-ups (EN, IO0, ESP-01 RST, GPIO2)
                 └─ SW3 throw "3V3"
SW3 common ── VIO ── U2 pin15 VIO (C4 100nF) + LED/RST pull-ups
```

- The "5 V" logic level is the +5V rail after the Schottky (~4.6–4.7 V). That is inside the CH343 VIO range (1.8–5 V) and is a valid high for 5 V AVRs.
- The CH343 V3 pin (pin 4) gets **only a 100 nF cap**. Do NOT tie it to VDD5 (VDD5 = 5 V here; tying V3 to VDD5 is only for 3.3 V supply).
- The SGM2212 (800 mA) handles ESP32 Wi-Fi peaks. Give the SOT-223 tab a large copper area.

### 3.2 CH343G (U2, SOP-16) pin map

From the WCH CH343 datasheet (LCSC C2846043 / C2844153). Pin numbers are the CH343G column.

| Pin | Name | Connect to |
|----|------|-----------|
| 1 | GND | GND |
| 2 | TXD | net **TXD** |
| 3 | RXD | net **RXD** |
| 4 | V3 | C3 100 nF to GND only |
| 5 | UD+ | USB D+ (via U3) |
| 6 | UD- | USB D- (via U3) |
| 7 | VBUS | VBUS_F (internal pull-down; USB-presence detect) |
| 8 | ACT# | LED4 (green) cathode; anode via R6 2.2k to VIO |
| 9 | DCD# | NC |
| 10 | TNOW / DTR# | net **DTR#**. **No pull-down resistor on this pin**: a 3–8k pull-down at power-up switches it to TNOW (RS485) mode. |
| 11 | RTS# | net **RTS#** |
| 12 | DSR# | NC |
| 13 | CTS# | NC (internal pull-up) |
| 14 | RI# | NC |
| 15 | VIO | net **VIO** + C4 100 nF |
| 16 | VDD5 | VBUS_F + C2 100 nF |

### 3.3 Auto-program (ESP32/ESP8266)

```
DTR# ── SJ1 ── DTR_AP ──┬── R7 10k ── Q1 base
                       └────────────── Q2 emitter
RTS# ── SJ2 ── RTS_AP ──┬── R8 10k ── Q2 base
                       └────────────── Q1 emitter
Q1 collector ── EN
Q2 collector ── IO0
```

esptool drives the classic sequence: DTR=0/RTS=1 → EN low (reset); DTR=1/RTS=0 → IO0 low while EN rises (boot mode). When DTR and RTS are both asserted (as a serial monitor does), the transistors stay off, so the board does not hold the target in reset.

MMBT3904 SOT-23 pinout: **1 = B, 2 = E, 3 = C**. Check the EasyEDA symbol-to-footprint pin map for both transistors before ordering.

### 3.4 Arduino reset

`DTR#` → C10 100 nF → net **RST** (H1 pin 8). R14 10k from RST to VIO.
Use this pin for Arduino Pro Mini (GRN/DTR pin also works), bare ATmega328P/168, LGT8F328P, etc.
The cap is independent of SJ1, so the Arduino reset still works when ESP auto-program is disabled.

### 3.5 UPDI mode

SW4 (SPDT, one throw unused): common → R15 4.7k → **RXD**; throw A → **TXD**; throw B → NC.
- SW4 = UART (default): R15 is disconnected and the board is a normal UART.
- SW4 = UPDI: TX → 4.7k → RX. Connect **RX pin → UPDI** of the target, plus GND and VCC. Set SW3 to the target's voltage.
- Arduino IDE: megaTinyCore/DxCore → Programmer "SerialUPDI - 230400 baud".

### 3.6 ESP-01 socket (J2, 2×4 female, 2.54 mm)

ESP-01 pinout (module component side up, antenna pointing away from the header):

```
 GND   GPIO2  GPIO0  RXD
 TXD   CH_PD  RST    VCC
```

| ESP-01 pin | Net |
|-----------|-----|
| VCC | +3V3 (C8 22 µF next to the socket) |
| GND | GND |
| TXD | **RXD** (to CH343 RXD) |
| RXD | **TXD** (from CH343 TXD) |
| CH_PD / EN | **EN** |
| RST | ESP01_RST: R12 10k to +3V3 |
| GPIO0 | **IO0** |
| GPIO2 | R13 10k to +3V3 |

Silkscreen next to J2: **"ESP-01: SET 3.3V!"** plus an antenna-direction outline. With SW3 on 5 V, the UART lines would be ~4.6 V, which is above the ESP8266 absolute maximum.
Check this pin order against your own ESP-01 module before placing J2. Clone modules occasionally differ.

### 3.7 Bare-module pull-ups (ESP-12F, ESP32-WROOM, ESP32-C3/S3 modules)

| Net | Part |
|-----|------|
| EN | R9 10k to +3V3, C9 100 nF to GND |
| IO0 | R10 10k to +3V3 |
| IO15 (H1 pin 11) | R11 10k to GND (needed by ESP8266; harmless on ESP32) |

ESP8266 GPIO2 must be high at boot (module-internal pull-up is normally enough). **Do not** add a GPIO2 pull-up on the main header, because ESP32 needs GPIO2 low/floating to enter download mode.
Dev boards (NodeMCU, DevKitC) have their own pull-ups. The extra 10k in parallel is harmless.

### 3.8 Buttons and LEDs

- SW1 RESET: EN → GND. SW2 BOOT: IO0 → GND (TS-1088, as V1).
- LED1 red PWR: +3V3 → R3 1k → LED → GND.
- LED2 orange TX: VIO → R4 2.2k → LED → TXD (lights when the line is low).
- LED3 yellow RX: VIO → R5 2.2k → LED → RXD.
- LED4 green ACT: VIO → R6 2.2k → LED → ACT# (on when USB is enumerated).

---

## 4. Main header H1 (1×11, 2.54 mm): silkscreen and colours

| Pin | Label | Net | Colour |
|----|-------|-----|--------|
| 1 | 3V3 | +3V3 | Red |
| 2 | 5V | +5V | Purple |
| 3 | GND | GND | Black |
| 4 | TX | TXD (out from flasher → target RX) | Orange |
| 5 | RX/UPDI | RXD (in to flasher ← target TX) | Green |
| 6 | DTR | DTR# (raw) | Blue |
| 7 | RTS | RTS# (raw) | Blue |
| 8 | RST | RST (DTR# via 100 nF) | Blue |
| 9 | EN | EN | Yellow |
| 10 | IO0 | IO0 | Yellow |
| 11 | IO15 | IO15 (10k to GND) | Grey |

### Wiring cheat-sheet (print on the back silkscreen)

| Target | Wires |
|--------|-------|
| ESP32 / ESP8266 dev board | Use the board's own USB, or 3V3·GND·TX→RX·RX←TX·EN·IO0 |
| Bare ESP-12F | 3V3·GND·TX·RX·EN·IO0·IO15 (SW3 = 3V3) |
| ESP-01 / 01S | Plug into J2 (SW3 = 3V3) |
| Arduino Pro Mini / bare 328P | VCC(5V or 3V3)·GND·TX·RX·**RST** (SW3 = target voltage) |
| ATtiny 1614 / AVR-DA (UPDI) | VCC·GND·**RX/UPDI** (SW4 = UPDI) |

---

## 5. Full netlist

See [`netlist.csv`](netlist.csv): one row per net, listing every `Ref.pin`. Use it to check the EasyEDA schematic net by net.

## 6. BOM

See [`BOM_V2_LCSC.csv`](BOM_V2_LCSC.csv). Every LCSC number was looked up on lcsc.com on 2026-11-12.
Notes:
- **CH343G (C2844153)**: only 500 pcs in LCSC stock, more from "other suppliers" (3–7 day lead time). If it runs out, the **CH343P (C2846043, QFN-16)** is the same die, but the pinout differs (see the datasheet) and it is not hand-solderable.
- No DPDT slide switch was in stock (the C&K SS22D07/SK22D07 families showed 0 stock). If one becomes available, the second pole of SW3 could drive a red "5V IO" warning LED.
- SJ1/SJ2 are PCB solder jumpers (EasyEDA "Solder Jumper, bridged" footprint), not BOM parts.

## 7. Layout guidance (for EasyEDA)

- Target size about 65 × 32 mm, 2 layers, 1.6 mm. Keep the H1 pin order and colour-coded silkscreen from V1.
- Place U3 (USBLC6) right at the USB-C connector. Route D+/D- as a short, parallel pair over unbroken GND, with no vias if possible. The CH343 is USB full-speed (12 Mbit/s), so over a few centimetres controlled impedance is not critical.
- C2/C3/C4 within 2 mm of U2 pins 16/4/15.
- SOT-223 tab: ≥ 1 cm² of copper on the top layer with vias to the bottom GND.
- Put J2 (ESP-01) at the board end opposite USB-C, with the antenna overhanging the edge. No copper under the module antenna area.
- SW3 and SW4 on the board edge with large silk labels: `IO: 3V3 | 5V` and `UART | UPDI`.
- Test pads: VBUS_F, +5V, +3V3, VIO, GND, TXD, RXD, DTR#, RTS#.

## 8. Bring-up test plan

1. Before fitting U1/U2: check for no short between VBUS, +5V, +3V3 and GND.
2. Power up: +5V ≈ 4.6–4.7 V, +3V3 = 3.30 V, VIO follows SW3. PWR LED on.
3. Plug into the PC. ACT LED on, COM port appears (Windows: "USB-SERIAL CH343" or "USB Serial Device").
4. Loopback: bridge TX↔RX and type in a terminal at 115200, then 2 Mbaud. TX/RX LEDs blink.
5. Scope or meter EN/IO0 while running `esptool.py --port COMx chip_id`: EN pulses low, then IO0 is held low as EN rises.
6. Flash: ESP32 DevKit (wired via header), bare ESP-12F (header + IO15), ESP-01 (J2), Arduino Pro Mini 5 V (SW3=5V, RST pin), ATtiny1614 (SW4=UPDI, megaTinyCore SerialUPDI).
7. Short the 5V pin to GND briefly: the polyfuse trips, and the PC port survives.
