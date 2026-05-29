# StratusLog — Schematic Review & Valve Driver Add-on

This document contains:
1. An expert review of the existing `stratuslog.kicad_sch` design
2. A complete valve-driver circuit (5 V→12 V boost is already present; this adds latching + non-latching valve outputs)
3. Step-by-step KiCad integration instructions

Hardware identified in the current design:

| Block | Part | Notes |
|---|---|---|
| MCU | ATmega1284P-A (TQFP-44) | Arduino-class AVR, 128 kB flash |
| RTC | DS3231M (SOIC-16W) | MEMS, ±5 ppm, no external XTAL needed |
| Storage | microSD (Hirose DM3AT-SF-PEJM5) | premium card slot |
| Comms | LSM100A (Murata) | LoRa/Sigfox, STM32WL inside |
| SDI-12 | SN74LVC1G240 + BSS123 + BZX84C7V5 | classic Vegetronix-style transceiver |
| Boost | LT1372HVCS8 (SOIC-8) | 5 V → 12 V — **incomplete**, see §1.1 |
| Sensor I/O | 3× Phoenix MKDS-1-7-3.81 (7-pos) | flexible per-channel breakout |
| Antenna | SMA (Amphenol 132203-12) | external whip |
| RTC backup | CR2032 holder | |
| Programming | J7 (1×5 header, 2.54 mm) | appears to be **LSM100A SWD**, not AVR ISP |

---

## 1. Findings & recommendations (ranked by severity)

### 🔴 Critical — fix before fabrication

#### 1.1 Boost converter (U4 LT1372) is missing required external parts
The LT1372HVCS8 needs the following parts to actually work as a 5 V→12 V boost. None of them are present in the current schematic:

| Part | Spec | Reason |
|---|---|---|
| **L1 — power inductor** | 22 µH, ≥ 2 A I_sat, low-DCR (e.g., **Würth 7447789004** or **Coilcraft MSS1038-223**) | Stores energy each switching cycle |
| **D1 — Schottky diode** | 1 A, 30 V, low Vf (e.g., **SS14**, **B130**, or **MBRA130T3G**) | Output rectifier |
| **C_OUT (12 V rail)** | 22 µF + 100 nF ceramic, 25 V (or 47 µF tantalum) | Output ripple — `C10 = 10 µF` alone is **not enough** for a 1.5 A switcher |
| **C_VC** | 0.022 µF + Rc 1 kΩ in series | Loop compensation on Vc pin (LT1372 is current-mode, requires this) |
| **R_FB1 / R_FB2** | 1.21 V ref → for 12 V out: **R_FB1 = 8.66 kΩ, R_FB2 = 1 kΩ** (`Vout = 1.21·(1+R1/R2)`) | Output voltage setting |
| **C_IN (5 V side)** | 22 µF ceramic + 100 nF | Input ripple, MUST be very close to LT1372 Vin pin |

LT1372 also needs the `S/S` (soft-start / shutdown) pin handled — pull to Vin via 100 kΩ for always-on, or drive from a 1284P GPIO for software enable (recommended; saves ~4 mA quiescent when 12 V isn't needed).

#### 1.2 No 3.3 V regulator visible — power tree is broken
The schematic has nets named `3V3`, `5V`, `5VOUT`, `12VOUT`, and `VIN`, but **no LDO or buck regulator** is in the BOM. The LSM100A, microSD, and (presumably) the ATmega1284P all need 3.3 V. The LSM100A's internal 3.3 V LDO is **not** rated to power external loads.

**Add one of:**
- **MCP1700-3302E/TT** (SOT-23, 250 mA, 1.6 µA Iq) — perfect for low-power loggers, drop-in if VIN ≤ 6 V
- **MCP1825S-3302E/DB** (SOT-223, 500 mA, 120 µA Iq) — if VIN can be up to 6 V and you want headroom for SD inrush
- **TPS62840** (buck, 750 mA, 60 nA Iq) — best efficiency for battery operation; SOT583 package

Place close to the LSM100A and add a 10 µF ceramic on its output.

#### 1.3 ATmega1284P has no ISP header
J7 is wired to RST/3V3/DIO/CLK/GND — that's **SWD for the LSM100A** (STM32WL), not AVR ISP. There is no path to flash the AVR's bootloader.

**Add:** A 2×3 1.27 mm or 2.54 mm ICSP header (`Connector_PinSocket_2x03_P2.54mm_Vertical`) wired:

| Pin | Signal | AVR pin |
|---|---|---|
| 1 | MISO | PB6 (TQFP pin 7) |
| 2 | VCC (3V3) | — |
| 3 | SCK | PB7 (TQFP pin 8) |
| 4 | MOSI | PB5 (TQFP pin 6) |
| 5 | RESET | PB4 not — actual RESET pin (TQFP pin 4) |
| 6 | GND | — |

Or, if board space is tight, expose those signals on a **tag-connect TC2030-IDC-NL** footprint (no connector required, programmer has spring pins).

#### 1.4 Crystal Y1 has no Value or Footprint set
The symbol shows `Value = "Crystal"` and an empty `Footprint`. This will fail BOM export and PCB DRC.

- At **3.3 V Vcc**, the ATmega1284P-A is rated **0–10 MHz**. Use **8.000 MHz** in HC-49UP-SMD or 3.2×2.5 mm SMD.
- At **5 V Vcc**, you can go up to 20 MHz. But your 3V3 net suggests 3.3 V operation → **8 MHz** is the safe Arduino-style choice (matches MiniCore "8 MHz external" board variant).

Set `Value = 8MHz` and `Footprint = Crystal:Crystal_SMD_HC49-SD` (or HC-49UP-SMD).

Also add **two load caps** (typically **18 pF or 22 pF** per the crystal datasheet) from each leg to GND. They're missing from your schematic.

#### 1.5 0201 resistor footprints chosen with "HandSolder" pads
Every R is `R_0201_0603Metric_Pad0.64x0.40mm_HandSolder`. **0201 imperial = 0.6 × 0.3 mm.** That is *not* hand-solderable, even with the slightly enlarged pads. The "HandSolder" suffix usually applies to 0603 imperial and up.

**Change all resistors to** `Resistor_SMD:R_0603_1608Metric_Pad1.05x0.95mm_HandSolder` (or 0805 to match the caps). One global replace in KiCad: Tools → Edit Symbol Fields Table → set Footprint column for all R*.

### 🟠 Important

#### 1.6 SDI-12 protection Zener is too high
D1 is `BZX84C7V5` (7.5 V). SDI-12 marking voltage is 5 V, but transients on a long field cable can be much higher. The SN74LVC1G240's absolute max input is **6.5 V**. A 7.5 V Zener won't clamp until *after* the buffer is already damaged.

**Replace with `BZX84C5V6` (5.6 V)** or, better, a unidirectional TVS like **PESD5V0S1BA** (low capacitance) or **SMAJ5.0A** (higher energy, larger footprint) for field robustness.

#### 1.7 No reverse-polarity protection on VIN
J5 is a simple 2-pin terminal block. Field installers will eventually reverse it.

**Add:** A P-channel MOSFET reverse-polarity block (e.g., **DMP3098L** or **AO3401**) — gate to GND through 100 kΩ, source to load, drain to J5. Or a Schottky in series (cheap but wastes ~0.3 V).

#### 1.8 No fuse / no TVS on VIN
For a fielded data logger this is a real reliability concern. Add:
- **PTC resettable fuse**: 0805 footprint, hold = 250 mA (e.g., **MF-PSMF010X**)
- **Bidirectional TVS**: **SMAJ12CA** (if VIN ≤ 12 V) or **SMAJ24CA** (if VIN up to 24 V)

#### 1.9 LSM100A — UART logic level mismatch risk
If the ATmega1284P is at **3.3 V** (consistent with the rest of your design), MCU↔LSM100A UART is fine — both at 3.3 V.

If the AVR ends up at **5 V**, the LSM100A UART RX pin is **not 5 V tolerant** on most STM32WL parts. You'd need a level shifter (e.g., **TXB0104**) on UART_TX and SWD lines.

Confirm Vcc(AVR) — this also gates the crystal frequency choice (§1.4).

#### 1.10 microSD card — 5 V-only loggers often die from inrush
The microSD slot needs a soft-start cap (≥ 22 µF on its 3.3 V power pin) and ideally a series ferrite. Check that there's a dedicated bulk cap right at J4's Vcc pin.

### 🟡 Nice-to-have

- **Power-good LEDs** for 3V3 and 12VOUT (D2/D3 currently have no footprint or net mapping documented).
- **Test points** on 3V3, 12VOUT, RST, UART_TX, UART_RX — invaluable when debugging in the field with a 'scope.
- **Silkscreen pin-1 marker** on every IC (default KiCad footprints have it; confirm).
- **Mounting hole 4 corners** are present (H1–H4) ✓ — good.
- **DS3231M I²C pull-ups (R7, R8 = 4.7 kΩ)** ✓ — good values.
- **Battery decoupling for DS3231M backup** (Vbat to GND 0.1 µF) — verify present.

---

## 2. Valve driver add-on (new sub-sheet `valves.kicad_sch`)

Adds three switched outputs onto the existing 12VOUT rail:
- **V1**: latching / pulse valve (single-coil polarity-reversal) via DRV8871 H-bridge
- **V2, V3**: non-latching solenoid valves via ULN2803A low-side switch
- **V_PWR_EN**: optional master power gate (P-MOSFET on the 12VOUT rail)

### 2.1 Block diagram

```
  12VOUT ────────────────────┬────────────────────────────────┐
                             │                                │
                    [VM] DRV8871DDA  (U_VLV1)         [COM]   │
                                                              │
   PA5 ── 1k ── IN1 ─►|             |─► OUT1 ──── J_VLV.1 (V1+)
   PA4 ── 1k ── IN2 ─►|             |─► OUT2 ──── J_VLV.2 (V1-)
                      | ISEN─Risen──|
                       33 kΩ        GND
                             │
                             │
                       ULN2803A (U_VLV2)
                             │
   PA6 ──────────── IN1 ─►|        |─► OUT1 ── J_VLV.3 (V2)
   PA7 ──────────── IN2 ─►|        |─► OUT2 ── J_VLV.5 (V3)
                          | COM ── 12VOUT
                          | GND ── GND
                          
  J_VLV (1×6 Phoenix MKDS-1-6-3.81): V1+, V1-, V2, GND, V3, GND
```

### 2.2 Bill of materials (additions)

| Ref | Value | Footprint | Notes |
|---|---|---|---|
| U_VLV1 | DRV8871DDA | SOIC-8 PowerPad (Package_SO:HSOP-8-1EP_3.9x4.9mm_P1.27mm_EP2.41x3.1mm) | TI dual-input H-bridge, 6.5–45 V, 3.6 A peak |
| U_VLV2 | ULN2803A | SOIC-18W (Package_SO:SOIC-18W_7.5x11.6mm_P1.27mm) | 8-ch Darlington array, internal flyback diodes |
| C_VLV1 | 10 µF / 25 V | C_0805 | DRV8871 VM bulk |
| C_VLV2 | 100 nF | C_0805 | DRV8871 VM HF decouple |
| C_VLV3 | 100 nF | C_0805 | ULN2803 VCC decouple |
| R_VLV1 | 33 kΩ ±1 % | R_0603 | DRV8871 ISEN — limits current to ~2 A (`I_lim = 4500 / Risen` mA) |
| R_VLV2 | 1 kΩ | R_0603 | PA5 → IN1 series |
| R_VLV3 | 1 kΩ | R_0603 | PA4 → IN2 series |
| R_VLV4 | 100 kΩ | R_0603 | IN1 pulldown (defines OFF on AVR reset) |
| R_VLV5 | 100 kΩ | R_0603 | IN2 pulldown |
| J_VLV | Phoenix MKDS-1-6-3.81 | TerminalBlock_Phoenix_MKDS-1-6-3.81_1x06_P3.81mm_Horizontal | matches your existing J1/J2/J3 series |

Optional master power gate (recommended — kills 12 V when valves not in use):

| Ref | Value | Footprint | Notes |
|---|---|---|---|
| Q_VLV1 | AO3401 (P-FET) | SOT-23 | 12V_SW = on when gate pulled low |
| Q_VLV2 | 2N7002 (N-FET) | SOT-23 | level-shifts 1284P GPIO to 12V P-FET |
| R_VLV6 | 10 kΩ | R_0603 | P-FET gate pull-up to 12VOUT |
| R_VLV7 | 1 kΩ | R_0603 | 1284P → 2N7002 gate |
| R_VLV8 | 100 kΩ | R_0603 | 2N7002 gate pulldown |

### 2.3 ATmega1284P pin assignment

| TQFP-44 pin | AVR | Hierarchical label | Function |
|---|---|---|---|
| 32 | PA5 | `V1_IN1` | DRV8871 IN1 — pulse OPEN |
| 33 | PA4 | `V1_IN2` | DRV8871 IN2 — pulse CLOSE |
| 31 | PA6 | `V2_EN` | ULN2803 IN1 — non-latching V2 ON/OFF |
| 30 | PA7 | `V3_EN` | ULN2803 IN2 — non-latching V3 ON/OFF |

> **Note:** TQFP-44 pin numbers for Port A are 30–37 (PA7=30 … PA0=37). The DIP-40 equivalents are PA4=36, PA5=35, PA6=34, PA7=33.
| (any free) | e.g., PB1 | `V_PWR_EN` | optional master 12 V gate |

### 2.4 Component sizing rationale

- **DRV8871 ISEN = 33 kΩ** → ~2.0 A current limit. Most pulse latching valves draw 0.8–1.5 A inrush for ~10 ms; this gives margin.
- **Pulse width (firmware): 50 ms** for typical irrigation latching valves; consult the valve datasheet (range 20–100 ms).
- **ULN2803 limit**: 500 mA per channel, ~1.5 A total package. If V2/V3 each draw < 250 mA holding current, you can run both simultaneously safely.
- **Boost converter sizing impact**: With both V2 and V3 on (say 200 mA each at 12 V) plus a V1 latching pulse, momentary load is ~1 A at 12 V → ~2.4 A from the 5 V rail. Verify L1 rating in §1.1 supports this.

### 2.5 Firmware skeleton (Arduino/MiniCore for ATmega1284P)

```c
#include <util/delay.h>

#define V1_IN1   PA5
#define V1_IN2   PA4
#define V2_EN    PA6
#define V3_EN    PA7
#define VPWR_EN  PB1   // optional master gate

#define PULSE_MS 50

void valves_init(void) {
    DDRA |= (1<<V1_IN1) | (1<<V1_IN2) | (1<<V2_EN) | (1<<V3_EN);
    DDRB |= (1<<VPWR_EN);
    PORTA &= ~((1<<V1_IN1)|(1<<V1_IN2)|(1<<V2_EN)|(1<<V3_EN));
    PORTB &= ~(1<<VPWR_EN);   // 12 V gated OFF on boot
}

void valves_power(uint8_t on) {
    if (on) PORTB |= (1<<VPWR_EN); else PORTB &= ~(1<<VPWR_EN);
    if (on) _delay_ms(2);     // let boost stabilize
}

void v1_open(void)  {
    PORTA = (PORTA & ~(1<<V1_IN2)) | (1<<V1_IN1);
    _delay_ms(PULSE_MS);
    PORTA &= ~((1<<V1_IN1)|(1<<V1_IN2));
}
void v1_close(void) {
    PORTA = (PORTA & ~(1<<V1_IN1)) | (1<<V1_IN2);
    _delay_ms(PULSE_MS);
    PORTA &= ~((1<<V1_IN1)|(1<<V1_IN2));
}
void v2_on(void)  { PORTA |=  (1<<V2_EN); }
void v2_off(void) { PORTA &= ~(1<<V2_EN); }
void v3_on(void)  { PORTA |=  (1<<V3_EN); }
void v3_off(void) { PORTA &= ~(1<<V3_EN); }
```

---

## 3. KiCad integration steps

This branch ships a starter file `valves.kicad_sch` (placeholder hierarchical labels only; populate components in KiCad).

In KiCad 9:

1. Open `stratuslog.kicad_pro`.
2. In Eeschema, **File → Append Hierarchical Sheet** (or **Place → Hierarchical Sheet**).
3. Choose **`valves.kicad_sch`** as the file. Place the sheet symbol in an empty area of the root schematic (right of U4 LT1372 is convenient).
4. Add the hierarchical pins listed in §2.3 (12VOUT, GND, V1_IN1, V1_IN2, V2_EN, V3_EN, optional V_PWR_EN). Their labels should match the parent schematic's existing nets.
5. Open the new sub-sheet (double-click) and lay down the components per §2.2 BOM and §2.1 block diagram.
6. Run **Inspect → Electrical Rules Checker (ERC)** to validate.
7. **Tools → Update PCB from Schematic** to push to the layout.

---

## 4. Suggested next steps (priority order)

1. Fix §1.1 (boost completion) and §1.2 (3.3 V LDO) — board does not power up otherwise.
2. Fix §1.3 (AVR ISP) — board cannot be flashed otherwise.
3. Fix §1.4 (crystal value) and §1.5 (0201 → 0603) — fab will fail or be unbuildable.
4. Add valve driver sub-sheet from §2 if your application needs valve actuation.
5. Apply §1.6–1.10 hardening before production run.
6. Run final ERC + DRC.


---

## 5. Programming headers for the ATmega1284P-A

The board currently has **no AVR programming header** (J7 is SWD for the LSM100A). Add the following.

### 5.1 AVR ISP / ICSP — 6-pin (required)

Standard Atmel AVR-ISP-6 (2×3, 2.54 mm). Used to burn the bootloader and/or flash directly.

```
        ┌─────────────┐
 MISO  1│ ●  ●        │2  VCC (3V3)
  SCK  3│ ●  ●        │4  MOSI
  RST  5│ ●  ●        │6  GND
        └─────────────┘
```

| Hdr pin | Signal | Port | TQFP-44 pin | Net | Description |
|---|---|---|---|---|---|
| 1 | MISO | PB6 | 2 | `MISO` | target → programmer |
| 2 | VCC | — | — | `3V3` | power/sense (3.3 V) |
| 3 | SCK | PB7 | 3 | `SCK` | ISP clock |
| 4 | MOSI | PB5 | 1 | `MOSI` | programmer → target |
| 5 | RESET | RESET | 4 | `RST` | low during programming |
| 6 | GND | — | — | `GND` | ground |

- Taps the existing microSD SPI bus (MOSI/MISO/SCK) + RESET. SD CS stays high during ISP → no conflict.
- Symbol `Connector:AVR-ISP-6`; footprint `Connector_PinHeader_2.54mm:PinHeader_2x03_P2.54mm_Vertical`.
- Tight-space option: Tag-Connect **TC2030-IDC-NL** (no connector).
- **Requires:** 10 kΩ pull-up on RESET to 3V3, and 100 nF on each VCC/AVCC (AVCC via ferrite/10 µH + 100 nF).

### 5.2 Arduino serial / FTDI — 6-pin (optional, after bootloader)

SparkFun "FTDI Basic" order. Uses USART0 (PD0/PD1).

```
 [GND][CTS][VCC][TXO][RXI][DTR]
   1    2    3    4    5    6
```

| Hdr pin | Silk | To AVR | Port | TQFP-44 pin | Net | Description |
|---|---|---|---|---|---|---|
| 1 | GND | GND | — | — | `GND` | ground |
| 2 | CTS | GND/NC | — | — | — | tie to GND |
| 3 | VCC | 3V3 | — | — | `3V3` | use a **3.3 V** FTDI cable |
| 4 | TXO | RXD0 | PD0 | 9 | `UART_RX` | FTDI → MCU RX |
| 5 | RXI | TXD0 | PD1 | 10 | `UART_TX` | MCU TX → FTDI |
| 6 | DTR | RESET via 100 nF | RESET | 4 | `RST` | auto-reset into bootloader |

- Auto-reset needs a **100 nF** series cap between DTR and RESET (plus the 10 kΩ RESET pull-up).

### 5.3 Toolchain / fuses (3.3 V)

- At 3.3 V, max clock is 10 MHz → run the external crystal at **8 MHz**.
- Arduino IDE: **MightyCore → ATmega1284 → External 8 MHz**, then **Burn Bootloader** once via ISP.
- Direct flash (USBasp): `avrdude -c usbasp -p m1284p -U flash:w:firmware.hex:i`
- After fuses select the external crystal, the chip needs Y1 present to start.


---

## 6. Tipping-bucket rain-gauge pulse input (PB2)

The `PULSE` header net should route to **PB2** (TQFP-44 pin 42), which is both **INT2** (hardware interrupt) and **PCINT10** (pin-change, wakes from Power-down). A tipping bucket is a reed switch that briefly shorts `PULSE`→`GND` on each tip; the design challenge is contact-bounce rejection.

### 6.1 Recommended circuit (RC + Schmitt + TVS)

```
            3V3
             │
            R1 10k          C2 100nF
             │               │
   PULSE  ●──┬──[R2 10k]──┬──┴── U_PC 74LVC1G17 (Schmitt, non-inv) ──● PB2
             │            │
          reed sw       C1 1uF
          (gauge)         │
             │           GND
   GND    ●──┴───────────────────────────────────────────────────── GND

   D_PC TVS (PESD3V3L1BA) across PULSE–GND; optional 100R / 600R ferrite at terminal
```

- Idle: reed open → node high → PB2 high. Tip: reed closed → node low → **falling edge** on PB2.
- τ = R2·C1 = **10 ms** debounce (reed bounce <1 ms; real tips are seconds apart even in extreme rain, so no risk of merging tips).
- 74LVC1G17 hysteresis guarantees a single clean edge from the slow RC ramp (critical when the MCU sleeps and isn't polling).

### 6.2 BOM

| Ref | Value | Footprint | Purpose |
|---|---|---|---|
| R1 | 10 kΩ | R_0603 | pull-up |
| R2 | 10 kΩ | R_0603 | RC series |
| C1 | 1 µF | C_0805 | RC cap (10 ms) |
| C2 | 100 nF | C_0603 | Schmitt decouple |
| U_PC | 74LVC1G17 | SOT-23-5 | non-inverting Schmitt buffer |
| D_PC | PESD3V3L1BA | SOD-523 | ESD/surge TVS |
| FB1 (opt) | 600 Ω ferrite | 0603 | EMI |

Budget variant: omit U_PC/C2, tie RC node to PB2, use internal pull-up + software debounce.

### 6.3 Firmware (register-level)

```c
volatile uint32_t tip_count = 0;
void pulse_init(void) {
    DDRB  &= ~(1<<PB2);     // input
    PORTB |=  (1<<PB2);     // internal pull-up
    EICRA |=  (1<<ISC21);   // INT2 falling edge
    EICRA &= ~(1<<ISC20);
    EIMSK |=  (1<<INT2);
    sei();
}
ISR(INT2_vect) { tip_count++; }
```

### 6.4 Sleep gotcha

Edge-triggered INT2 does **not** wake the ATmega1284P from Power-down (I/O clock stopped). To wake on a tip, either:
- use **PCINT10 on PB2** (`PCICR|=(1<<PCIE1); PCMSK1|=(1<<PCINT10);`, count in `ISR(PCINT1_vect)`) — wakes on any edge, or
- use **INT2 low-level mode** (`ISC2=00`) — wakes from Power-down while line held low.
