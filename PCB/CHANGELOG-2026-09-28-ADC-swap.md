# Schematic changelog — ADC swap AD7606 → ADS131M08 (2026-09-28)

Baseline: commit `d215e9c` ("Changed power, started changing ADC and refactoring signal amplitude").
Files touched: `FOGLIO1.kicad_sch`, `FOGLIO2.kicad_sch`, `PCB v2.kicad_sch`. **The PCB (`PCB v2.kicad_pcb`) was not touched.**

Verification:
- the netlist was exported with `kicad-cli` before and after, and the full net diff is in section 7;
- ERC was run before and after (section 8).

Every net listed below comes from the exported netlist. None of it is transcribed by hand.

---

## 0. Edits you did not explicitly ask for (both are reversible)

1. **FOGLIO1 paper size A4 → A3.**
   - The new divider and LDO blocks needed space.
   - All existing items keep their coordinates, and the title block moves to the new bottom-right corner.
   - To go back to A4, you would need to move the two new blocks (x ≥ 294 mm) into free space first.
2. **Your off-page U2/U3 group was moved onto the sheet**, into the old AD7606 area.
   - Shift: +187.96 mm in X for U2, C35, C36, C37, the AIN labels and their wires and power symbols. U3 and its wiring got a further +12.7 mm, to make room for the SPI labels.
   - This is a pure translation, with no net change: I checked the netlist diff after this step.
   - One wire was removed: the direct U3.4 → U2.23 wire. It is replaced by R80 (see section 2).

## 1. Removed (AD7606 and parts used only by it)

| Ref | Value / footprint | Was connected to |
|---|---|---|
| U5 | AD7606, LQFP-64 | +5V (AVCC), GND, Vdrive, CONVST, RESET, SPI_SCK, SPI_CS, BUSY, SPI_MISO (the V1–V8 inputs were already disconnected at `d215e9c`) |
| C29, C30, C31 | 100n 0603 | U5 AVCC decoupling (+5V/GND) |
| C32 | 100n 0603 | U5 Vdrive decoupling |
| C45 | 1u 0603 | U5 REGCAP |
| C46 | 10u 0805 | U5 REFIN/REFOUT |
| C47 | 10u 0805 | U5 REFCAPA/B |
| D3, R8 | LED, 4.7k | BUSY LED |
| J7 (old) | Conn_01x07, `PinHeader_1x07…SMD_Pin1Right` | RESET, BUSY, SPI_SCK, SPI_CS, SPI_MISO, CONVST, Vdrive (**replaced**, see section 2) |

Also removed, because they were connected only to the parts above:
- 22 wires, 16 no-connect flags and 7 junctions;
- labels `Vdrive`, `BUSY`, `CONVST`, `RESET`, `SPI_SCK`, `SPI_CS`, `SPI_MISO`;
- 5× `+5V` and 14× `GND` power symbols;
- `#FLG03`, the PWR_FLAG that sat on Vdrive;
- the "BUSY LED CHECK" text and its dashed box, and the old J7 description text.

Checked:
- `+5V` lost only U5.1/37/38/48 and C29–C31. It still has C21, C33, C34, J2.A9/B9, R7, R11, R12, TP1 and U4.3.
- `GND` lost exactly the 22 pins of the deleted parts.

## 2. Added parts (37)

Coordinates are in mm on FOGLIO1. The pin→net column is the final exported netlist.

| Ref | Value | Footprint | Pin → net | Position | Why |
|---|---|---|---|---|---|
| C59 | 100n | `Capacitor_SMD:C_0603_1608Metric` | 1=Net-(U2-REFIN), 2=GND | (149.86, 121.92) | REFIN decoupling to AGND (U2.14). TI characterises noise/DR with 100 nF on REFIN when the internal 1.2 V reference is used. |
| R80 | 33R | `Resistor_SMD:R_0603_1608Metric` | 1=Net-(U2-XTAL1{slash}CLKIN), 2=Net-(R80-Pad2) | (160.02, 139.7) | Series/source termination U3.4 (Schmitt buffer out) -> U2.23 CLKIN. Place next to U3 (TI §11.1). |
| C58 | 100n | `Capacitor_SMD:C_0603_1608Metric` | 1=+3V3, 2=GND | (186.69, 127.0) | U3 (74LVC1G17) VCC decoupling, +3V3 to GND. |
| R79 | 100k | `Resistor_SMD:R_0603_1608Metric` | 1=MCU_CLK, 2=GND | (191.77, 144.78) | MCU_CLK pull-down at U3 input (keeps the clock input defined when J7 is unplugged). |
| R61 | 1.1k | `Resistor_SMD:R_0603_1608Metric` | 1=Vout1, 2=AIN0P | (309.88, 50.8) | Divider top, Vout1 -> AIN0P (ratio 1.0/2.1 = 0.476, ±2.45 V -> ±1.17 V). |
| R62 | 1k | `Resistor_SMD:R_0603_1608Metric` | 1=AIN0P, 2=GND | (317.5, 55.88) | Divider bottom, AIN0P -> GND. |
| C48 | 10n C0G | `Capacitor_SMD:C_0603_1608Metric` | 1=AIN0P, 2=GND | (325.12, 55.88) | Anti-alias cap AIN0P -> GND (with 524 Ω source: fc ≈ 30 kHz). C0G per TI §11.1. |
| R63 | 1.1k | `Resistor_SMD:R_0603_1608Metric` | 1=Vout2, 2=AIN1P | (309.88, 66.04) | Divider top, Vout2 -> AIN1P (ratio 1.0/2.1 = 0.476, ±2.45 V -> ±1.17 V). |
| R64 | 1k | `Resistor_SMD:R_0603_1608Metric` | 1=AIN1P, 2=GND | (317.5, 71.12) | Divider bottom, AIN1P -> GND. |
| C49 | 10n C0G | `Capacitor_SMD:C_0603_1608Metric` | 1=AIN1P, 2=GND | (325.12, 71.12) | Anti-alias cap AIN1P -> GND (with 524 Ω source: fc ≈ 30 kHz). C0G per TI §11.1. |
| R65 | 1.1k | `Resistor_SMD:R_0603_1608Metric` | 1=Vout3, 2=AIN2P | (309.88, 81.28) | Divider top, Vout3 -> AIN2P (ratio 1.0/2.1 = 0.476, ±2.45 V -> ±1.17 V). |
| R66 | 1k | `Resistor_SMD:R_0603_1608Metric` | 1=AIN2P, 2=GND | (317.5, 86.36) | Divider bottom, AIN2P -> GND. |
| C50 | 10n C0G | `Capacitor_SMD:C_0603_1608Metric` | 1=AIN2P, 2=GND | (325.12, 86.36) | Anti-alias cap AIN2P -> GND (with 524 Ω source: fc ≈ 30 kHz). C0G per TI §11.1. |
| R67 | 1.1k | `Resistor_SMD:R_0603_1608Metric` | 1=Vout4, 2=AIN3P | (309.88, 96.52) | Divider top, Vout4 -> AIN3P (ratio 1.0/2.1 = 0.476, ±2.45 V -> ±1.17 V). |
| R68 | 1k | `Resistor_SMD:R_0603_1608Metric` | 1=AIN3P, 2=GND | (317.5, 101.6) | Divider bottom, AIN3P -> GND. |
| C51 | 10n C0G | `Capacitor_SMD:C_0603_1608Metric` | 1=AIN3P, 2=GND | (325.12, 101.6) | Anti-alias cap AIN3P -> GND (with 524 Ω source: fc ≈ 30 kHz). C0G per TI §11.1. |
| R69 | 1.1k | `Resistor_SMD:R_0603_1608Metric` | 1=Vout5, 2=AIN4P | (309.88, 111.76) | Divider top, Vout5 -> AIN4P (ratio 1.0/2.1 = 0.476, ±2.45 V -> ±1.17 V). |
| R70 | 1k | `Resistor_SMD:R_0603_1608Metric` | 1=AIN4P, 2=GND | (317.5, 116.84) | Divider bottom, AIN4P -> GND. |
| C52 | 10n C0G | `Capacitor_SMD:C_0603_1608Metric` | 1=AIN4P, 2=GND | (325.12, 116.84) | Anti-alias cap AIN4P -> GND (with 524 Ω source: fc ≈ 30 kHz). C0G per TI §11.1. |
| R71 | 1.1k | `Resistor_SMD:R_0603_1608Metric` | 1=Vout6, 2=AIN5P | (309.88, 127.0) | Divider top, Vout6 -> AIN5P (ratio 1.0/2.1 = 0.476, ±2.45 V -> ±1.17 V). |
| R72 | 1k | `Resistor_SMD:R_0603_1608Metric` | 1=AIN5P, 2=GND | (317.5, 132.08) | Divider bottom, AIN5P -> GND. |
| C53 | 10n C0G | `Capacitor_SMD:C_0603_1608Metric` | 1=AIN5P, 2=GND | (325.12, 132.08) | Anti-alias cap AIN5P -> GND (with 524 Ω source: fc ≈ 30 kHz). C0G per TI §11.1. |
| R73 | 1.1k | `Resistor_SMD:R_0603_1608Metric` | 1=Vout7, 2=AIN6P | (309.88, 142.24) | Divider top, Vout7 -> AIN6P (ratio 1.0/2.1 = 0.476, ±2.45 V -> ±1.17 V). |
| R74 | 1k | `Resistor_SMD:R_0603_1608Metric` | 1=AIN6P, 2=GND | (317.5, 147.32) | Divider bottom, AIN6P -> GND. |
| C54 | 10n C0G | `Capacitor_SMD:C_0603_1608Metric` | 1=AIN6P, 2=GND | (325.12, 147.32) | Anti-alias cap AIN6P -> GND (with 524 Ω source: fc ≈ 30 kHz). C0G per TI §11.1. |
| R75 | 1.1k | `Resistor_SMD:R_0603_1608Metric` | 1=Vsac, 2=AIN7P | (309.88, 157.48) | Divider top, Vsac -> AIN7P (ratio 1.0/2.1 = 0.476, ±2.45 V -> ±1.17 V). |
| R76 | 1k | `Resistor_SMD:R_0603_1608Metric` | 1=AIN7P, 2=GND | (317.5, 162.56) | Divider bottom, AIN7P -> GND. |
| C55 | 10n C0G | `Capacitor_SMD:C_0603_1608Metric` | 1=AIN7P, 2=GND | (325.12, 162.56) | Anti-alias cap AIN7P -> GND (with 524 Ω source: fc ≈ 30 kHz). C0G per TI §11.1. |
| U7 | TPS7A2033PDBVR | `Package_TO_SOT_SMD:SOT-23-5` | 1/IN_1=+5V, 2/GND_2=GND, 3/EN_3=+5V, 4/NC_4=unconnected-(U7-NC-Pad4), 5/OUT_5=+3.3VA | (325.12, 205.74) | 3.3 V LDO for the ADC: IN=+5V, EN=+5V, OUT=+3.3VA (AVDD). 1.7 V headroom -> full PSRR (92 dB @1 kHz typ). |
| C60 | 1u | `Capacitor_SMD:C_0603_1608Metric` | 1=+5V, 2=GND | (312.42, 208.28) | U7 input capacitor (TPS7A20 CIN 1 µF). |
| C61 | 1u | `Capacitor_SMD:C_0603_1608Metric` | 1=+3.3VA, 2=GND | (340.36, 208.28) | U7 output capacitor (TPS7A20 COUT ≥ 1 µF, X7R). |
| FB1 | BLM18PG221SN1 | `Inductor_SMD:L_0603_1608Metric` | 1=+3.3VA, 2=+3V3 | (355.6, 203.2) | Ferrite bead +3.3VA -> +3V3: keeps DVDD/U3/pull-up switching current off AVDD. |
| J7 | Conn_02x06_Odd_Even | `Connector_PinHeader_2.54mm:PinHeader_2x06_P2.54mm_Vertical_SMD` | 1/Pin_1_1=GND, 2/Pin_2_2=MCU_CLK, 3/Pin_3_3=GND, 4/Pin_4_4=ADC_SCLK, 5/Pin_5_5=ADC_MOSI, 6/Pin_6_6=GND, 7/Pin_7_7=ADC_MISO, 8/Pin_8_8=ADC_CS, 9/Pin_9_9=ADC_DRDY, 10/Pin_10_10=ADC_SYNC_RESET, 11/Pin_11_11=GND, 12/Pin_12_12=GND | (236.22, 132.08) | Replaces the old 1x07 header. GND next to CLK and SCLK; 1,3,6,11,12 = GND. |
| R77 | 10k | `Resistor_SMD:R_0603_1608Metric` | 1=+3V3, 2=ADC_CS | (270.51, 132.08) | Pull-up ADC_CS -> +3V3 (active-low input stays deasserted with J7 unplugged). |
| R78 | 10k | `Resistor_SMD:R_0603_1608Metric` | 1=+3V3, 2=ADC_SYNC_RESET | (283.21, 132.08) | Pull-up ADC_SYNC_RESET -> +3V3 (active-low input stays deasserted with J7 unplugged). |
| R81 | 33R | `Resistor_SMD:R_0603_1608Metric` | 1=U2_DOUT, 2=ADC_MISO | (228.6, 160.02) | Source termination on U2.20 DOUT -> ADC_MISO. Schematic sits by J7, but PLACE IT NEXT TO U2 on the PCB. |
| R82 | 33R | `Resistor_SMD:R_0603_1608Metric` | 1=U2_DRDY, 2=ADC_DRDY | (261.62, 160.02) | Source termination on U2.18 DRDY -> ADC_DRDY. Schematic sits by J7, but PLACE IT NEXT TO U2 on the PCB. |

Notes on the new parts:
- **U7 TPS7A2033PDBVR**: I confirmed from the TI datasheet ordering table that this is an active, production part.
  - DBV pinout: IN=1, GND=2, EN=3, NC=4, OUT=5. This matches the KiCad `TPS7A20xxxDBV` symbol.
  - The KiCad symbol inherits from `LP5907MFX-1.2`. I embedded it flattened into the sheet, so it renders without needing the parent.
  - Pin 4 (NC) is left open. It has no internal connection.
  - Datasheet: CIN 1 µF, COUT ≥ 1 µF.
- **FB1**: value `BLM18PG221SN1` (Murata, 220 Ω @ 100 MHz, ≥1 A) is my suggestion. Swap it for whatever you picked in SimSurfing.
- **R81/R82** (33 Ω source terminations on DOUT/DRDY) are *drawn* in the OUTPUT PINS box to keep U2 readable, but they must be **placed next to U2** on the PCB. **R80** likewise goes next to U3's output (TI §11.1: "a source-termination resistor placed at the clock buffer").
- **C48–C55** have value `10n C0G`. TI §11.1 says "Use C0G capacitors on the analog inputs". Order C0G/NP0 parts.

## 3. Modified existing items

| Item | Change |
|---|---|
| U2 ADS131M08 (yours) | Newly connected: pin 14 REFIN → C59; 15 AVDD → **+3.3VA**; 26 DVDD → **+3V3**; 16 SYNC/RESET → `ADC_SYNC_RESET`; 17 CS → `ADC_CS`; 18 DRDY → `U2_DRDY` (→R82→`ADC_DRDY`); 19 SCLK → `ADC_SCLK`; 20 DOUT → `U2_DOUT` (→R81→`ADC_MISO`); 21 DIN → `ADC_MOSI`; 22 XTAL2 → no-connect flag; 23 CLKIN → R80. AINxN and CAP (C35) are unchanged. |
| U2 AIN0P…AIN7P | The 8 hierarchical labels that sat directly on these pins (`Vout1…Vout7`, `Vsac`) were converted **in place** to local labels `AIN0P…AIN7P`. New hierarchical labels `Vout1…Vout7`, `Vsac` now feed the divider rows. |
| C37 (1u) / C36 (1u) | Now on +3.3VA / +3V3; before, only on `Net-(U2-AVDD)` / `Net-(U2-DVDD)`. Values unchanged: they match TI §10.3, "AVDD and DVDD must each be decoupled with a 1-µF capacitor". |
| U3 74LVC1G17 (yours) | Footprint was empty → `Package_TO_SOT_SMD:SOT-23-5` (DBV pinout 1 NC, 2 A, 3 GND, 4 Y, 5 VCC, matching your symbol). |
| FOGLIO2 (SENSORS) | The 9 local labels `V_sac` were renamed `Vsac`. The one on U13 pin 10 (OUT_C, 233.68/116.84) became a hierarchical label `Vsac` (output). The net is unchanged: C8.1, R10.1, R25/R30/R33/R35/R43/R45/R55 pin 2, U13.10, plus the new R75.1. |
| Root sheet | New sheet pin `Vsac`: SENSORS sheet at (151.13, 90.17), output; POWER/ADC sheet at (40.64, 97.79), input. The two pins are joined by a **plain wire** routed under the sheets, not global labels: a global `Vsac` would clash with the local `Vsac` labels (ERC `same_local_global_label`). The netlist name is now `/SENSORS/Vsac`. |
| Dashed boxes | ADC CONVERTER box redrawn to fit U2/U3 (x 104–208). OUTPUT PINS box extended down to y 198. New boxes: "ADC INPUT DIVIDERS + ANTI-ALIAS" and "ADC SUPPLY (LDO + FERRITE)". |
| Texts | New J7/firmware note in OUTPUT PINS; notes under the divider and supply blocks. |
| PWR_FLAG | `#FLG04` on +3V3 (after FB1) and `#FLG05` on GND. GND had no PWR_FLAG in the baseline: a pre-existing ERC error, now fixed. |

## 4. Resulting circuit (summary)

- **Per channel, n = 0…7:** `Vout(n+1)` / `Vsac` → 1.1k → `AINnP` → (1k ∥ 10 nF C0G) to GND.
  - Ratio 0.476, so ±2.45 V becomes ±1.17 V.
  - Source impedance 524 Ω; anti-alias corner about 30 kHz.
  - AINnN is tied to GND.
  - **CH7 = Vsac**, the reference sensor, recorded for diagnostics.
- **Supply:** `+5V` → U7 (EN tied to IN) → `+3.3VA` (U2 AVDD, C37) → FB1 → `+3V3` (U2 DVDD/C36, U3 VCC/C58, R77/R78 pull-ups).
- **Clock:** J7.2 `MCU_CLK` (100k pull-down R79) → U3 Schmitt buffer → R80 33 Ω → U2 CLKIN. XTAL2 has a no-connect flag.
- **J7 (2×6), pin:**

  | Pin | Signal | Pin | Signal |
  |---|---|---|---|
  | 1 | GND | 2 | MCU_CLK |
  | 3 | GND | 4 | ADC_SCLK |
  | 5 | ADC_MOSI | 6 | GND |
  | 7 | ADC_MISO | 8 | ADC_CS |
  | 9 | ADC_DRDY | 10 | ADC_SYNC_RESET |
  | 11 | GND | 12 | GND |

  - Pin 12 is GND. My earlier proposal had "3V3_IO sense" there; it isn't needed.
  - `ADC_CS` and `ADC_SYNC_RESET` have 10k pull-ups to +3V3.

## 5. Firmware-relevant defaults (from the note on the sheet)

- **Clock:** 5.12 MHz on MCO2 (PC9), 40–60 % duty. Use an even MCO divider.
- **ADC config:** OSR 256 → 10 kSPS. Set `XTAL_DIS=1` in the CLOCK register. WLENGTH = 32-bit sign-extended.
- **SPI:** mode 1 (CPOL 0, CPHA 1), about 8 MHz. Derive the SPI clock from the same PLL as MCO2.
- **Reads:** on each DRDY falling edge, read a full 10-word frame (status + 8 channels + CRC slot).

## 6. Not done / yours to decide

- **Run "Update PCB from Schematic" in KiCad.** The update dialog will also show these net renames on existing pads:
  - `/SENSORS/V_sac` → `/SENSORS/Vsac`
  - `Net-(U2-AVDD)` → `+3.3VA`
  - `Net-(U2-DVDD)` → `+3V3`
  - `Vout1…Vout7` now end at R61/R63/…/R73 instead of the U2 AIN pads. Any tracks already routed from the op-amp outputs to U2's AIN pads will be flagged by DRC as wrong-net; reroute them through the dividers.
  - U5, C29–C32, C45–C47, D3, R8 and the old J7 footprint will be removed.
  - About 37 new footprints appear unplaced.
  - J7 changes footprint to `PinHeader_2x06_P2.54mm_Vertical_SMD`. I kept SMD to match your previous header; a THT header is mechanically stronger for a cable.
- **Parts that need attention on the PCB:**
  - R80/R81/R82 must go next to U3/U2.
  - C48–C55 go close to the U2 AIN pins.
  - C59 goes at REFIN.
  - Route digital traces away from the analog inputs (TI §11.1).
- ~~USB input filtering~~ → **done in Part 4** (D1 SMF5.0A + C62/FB2 pi filter).
- ~~U13 section D floating~~ → **done in Part 2**: the spare is now U12-D, wired as a follower with its + input at GND.
- **Open design questions from the review:**
  - ~~moving the charge amps into U6/U13 (the section swap)~~ → **done in Part 2**;
  - ~~the first-stage 1 MΩ ∥ 15 pF corner~~ → **confirmed intentional**: the stage runs as a transimpedance amp below the 10.6 kHz pole, values kept;
  - the expected sensor amplitude, which sets the gains.
- **Pre-existing ERC warnings left as they were:**
  - custom-library symbol/footprint link issues;
  - `vout3`/`vout4` global vs `Vout3`/`Vout4` hierarchical names;
  - `S8` local vs global;
  - 16 "similar labels".

## 7. Net diff (before → after, from `kicad-cli sch export netlist`)

Nets whose members changed; `unconnected-*` pseudo-nets are omitted. Unconnected pins went from 50 (mostly U5/U2) to 14, all intentional:
- op-amp NC pins;
- U13 section D (see section 6);
- U2 XTAL2;
- U3 NC;
- U7 NC.

- `+3.3VA`
  - before: —
  - after:  (5): C37.1 C61.1 FB1.1 U2.15 U7.5
- `+3V3`
  - before: (1): U3.5
  - after:  (7): C36.1 C58.1 FB1.2 R77.1 R78.1 U2.26 U3.5
- `+5V`
  - before: (17): C21.1 C29.2 C30.2 C31.2 C33.1 C34.1 J2.A9 J2.B9 R11.1 R12.1 R7.1 TP1.1 U4.3 U5.1 U5.37 U5.38 U5.48
  - after:  (13): C21.1 C33.1 C34.1 C60.1 J2.A9 J2.B9 R11.1 R12.1 R7.1 TP1.1 U4.3 U7.1 U7.3
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/ADC_CS`
  - before: —
  - after:  (3): J7.8 R77.2 U2.17
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/ADC_DRDY`
  - before: —
  - after:  (2): J7.9 R82.2
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/ADC_MISO`
  - before: —
  - after:  (2): J7.7 R81.2
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/ADC_MOSI`
  - before: —
  - after:  (2): J7.5 U2.21
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/ADC_SCLK`
  - before: —
  - after:  (2): J7.4 U2.19
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/ADC_SYNC_RESET`
  - before: —
  - after:  (3): J7.10 R78.2 U2.16
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/AIN0P`
  - before: —
  - after:  (4): C48.1 R61.2 R62.1 U2.29
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/AIN1P`
  - before: —
  - after:  (4): C49.1 R63.2 R64.1 U2.32
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/AIN2P`
  - before: —
  - after:  (4): C50.1 R65.2 R66.1 U2.1
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/AIN3P`
  - before: —
  - after:  (4): C51.1 R67.2 R68.1 U2.4
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/AIN4P`
  - before: —
  - after:  (4): C52.1 R69.2 R70.1 U2.5
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/AIN5P`
  - before: —
  - after:  (4): C53.1 R71.2 R72.1 U2.8
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/AIN6P`
  - before: —
  - after:  (4): C54.1 R73.2 R74.1 U2.9
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/AIN7P`
  - before: —
  - after:  (4): C55.1 R75.2 R76.1 U2.12
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/BUSY`
  - before: (3): J7.2 R8.1 U5.14
  - after:  —
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/CONVST`
  - before: (3): J7.6 U5.10 U5.9
  - after:  —
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/MCU_CLK`
  - before: (1): U3.2
  - after:  (3): J7.2 R79.1 U3.2
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/RESET`
  - before: (2): J7.1 U5.11
  - after:  —
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/SPI_CS`
  - before: (2): J7.4 U5.13
  - after:  —
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/SPI_MISO`
  - before: (2): J7.5 U5.24
  - after:  —
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/SPI_SCK`
  - before: (2): J7.3 U5.12
  - after:  —
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/U2_DOUT`
  - before: —
  - after:  (2): R81.1 U2.20
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/U2_DRDY`
  - before: —
  - after:  (2): R82.1 U2.18
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/Vdrive`
  - before: (6): C32.2 J7.7 U5.23 U5.34 U5.6 U5.7
  - after:  —
- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/Vsac`
  - before: (1): U2.12
  - after:  —
- `/SENSORS/V_sac`
  - before: (10): C8.1 R10.1 R25.2 R30.2 R33.2 R35.2 R43.2 R45.2 R55.2 U13.10
  - after:  —
- `/SENSORS/Vsac`
  - before: —
  - after:  (11): C8.1 R10.1 R25.2 R30.2 R33.2 R35.2 R43.2 R45.2 R55.2 R75.1 U13.10
- `GND`
  - before: (90): C11.1 C17.2 C19.1 C20.2 C21.2 C22.2 C23.2 C24.1 C25.2 C26.1 C27.2 C28.1 C29.1 C30.1 C31.1 C32.1 C33.2 C34.2 C35.2 C36.2 C37.2 C45.2 C46.2 C47.2 C7.1 C9.2 D2.1 D3.1 H1.1 H2.1 H3.1 H4.1 J2.A12 J2.B12 J2.SH J3.1 J3.10 J3.Z1 J3.Z2 J5.1 R26.1 R3.2 R31.1 R34.1 R36.1 R4.2 R44.1 R46.1 R56.1 R6.2 R9.2 TP5.1 U1.3 U11.12 U11.3 U12.12 U12.3 U13.12 U13.3 U2.10 U2.11 U2.13 U2.2 U2.25 U2.27 U2.28 U2.3 U2.30 U2.31 U2.6 U2.7 U3.3 U4.13 U4.4 U5.2 U5.26 U5.3 U5.35 U5.4 U5.40 U5.41 U5.43 U5.46 U5.47 U5.5 U5.63 U5.64 U5.8 U6.12 U6.3
  - after:  (95): C11.1 C17.2 C19.1 C20.2 C21.2 C22.2 C23.2 C24.1 C25.2 C26.1 C27.2 C28.1 C33.2 C34.2 C35.2 C36.2 C37.2 C48.2 C49.2 C50.2 C51.2 C52.2 C53.2 C54.2 C55.2 C58.2 C59.2 C60.2 C61.2 C7.1 C9.2 D2.1 H1.1 H2.1 H3.1 H4.1 J2.A12 J2.B12 J2.SH J3.1 J3.10 J3.Z1 J3.Z2 J5.1 J7.1 J7.11 J7.12 J7.3 J7.6 R26.1 R3.2 R31.1 R34.1 R36.1 R4.2 R44.1 R46.1 R56.1 R6.2 R62.2 R64.2 R66.2 R68.2 R70.2 R72.2 R74.2 R76.2 R79.2 R9.2 TP5.1 U1.3 U11.12 U11.3 U12.12 U12.3 U13.12 U13.3 U2.10 U2.11 U2.13 U2.2 U2.25 U2.27 U2.28 U2.3 U2.30 U2.31 U2.6 U2.7 U3.3 U4.13 U4.4 U6.12 U6.3 U7.2
- `Net-(C45-Pad1)`
  - before: (3): C45.1 U5.36 U5.39
  - after:  —
- `Net-(D3-A)`
  - before: (2): D3.2 R8.2
  - after:  —
- `Net-(R80-Pad2)`
  - before: —
  - after:  (2): R80.2 U3.4
- `Net-(U2-AVDD)`
  - before: (2): C37.1 U2.15
  - after:  —
- `Net-(U2-DVDD)`
  - before: (2): C36.1 U2.26
  - after:  —
- `Net-(U2-REFIN)`
  - before: —
  - after:  (2): C59.1 U2.14
- `Net-(U2-XTAL1{slash}CLKIN)`
  - before: (2): U2.23 U3.4
  - after:  (2): R80.1 U2.23
- `Net-(U5-REFCAPA)`
  - before: (3): C47.1 U5.44 U5.45
  - after:  —
- `Net-(U5-REFIN{slash}REFOUT)`
  - before: (2): C46.1 U5.42
  - after:  —
- `Vout1`
  - before: (3): R24.2 U2.29 U6.7
  - after:  (3): R24.2 R61.1 U6.7
- `Vout2`
  - before: (3): R29.2 U2.32 U6.16
  - after:  (3): R29.2 R63.1 U6.16
- `Vout3`
  - before: (3): R39.2 U11.7 U2.1
  - after:  (3): R39.2 R65.1 U11.7
- `Vout4`
  - before: (3): R38.2 U11.16 U2.4
  - after:  (3): R38.2 R67.1 U11.16
- `Vout5`
  - before: (3): R49.2 U12.7 U2.5
  - after:  (3): R49.2 R69.1 U12.7
- `Vout6`
  - before: (3): R48.2 U12.16 U2.8
  - after:  (3): R48.2 R71.1 U12.16
- `Vout7`
  - before: (3): R59.2 U13.7 U2.9
  - after:  (3): R59.2 R73.1 U13.7

## 8. ERC (`kicad-cli sch erc --severity-all`)

| Check | Before (`d215e9c`) | After |
|---|---|---|
| **Errors (total)** | **48** | **0** |
| pin_not_connected | 22 | 0 |
| pin_not_driven | 21 | 0 |
| power_pin_not_driven | 4 | 0 |
| hier_label_mismatch | 1 | 0 |
| isolated_pin_label | 2 | 0 |
| lib_symbol_mismatch | 12 | 8 (all pre-existing, on C11/C22–C28 in SENSORS; the 4 removed were C29–C32). None of the new parts are affected. |
| footprint_link_issues / lib_symbol_issues | 5 / 5 | 5 / 5 (pre-existing: J3 and OPA4350 custom libs) |
| multiple_net_names | 2 | 2 (pre-existing vout3/vout4) |
| same_local_global_label | 1 | 1 (pre-existing S8) |
| similar_labels | 16 | 16 (pre-existing) |

## Datasheet references

- ADS131M08 (SBAS950B):
  - AVDD/DVDD 1 µF (§10.3);
  - CAP 220 nF when DVDD > 2.7 V (§10.1);
  - REFIN 100 nF is the characterisation condition (spec-table preamble), not a requirement;
  - C0G on analog inputs and source termination at the clock buffer (§11.1);
  - fCLKIN range 0.3–8.4 MHz in HR mode, gain ≤ 2 (Rec. Op. Cond.);
  - OSR options (CLOCK register).
- TPS7A20 (TI datasheet): DBV pinout, CIN 1 µF, COUT 1–200 µF, TPS7A2033PDBVR active.


---

# Part 2: op-amp section swap + spare tie-off (FOGLIO2 / SENSORS)

Only `FOGLIO2.kicad_sch` changed. All 56 existing symbols were **re-positioned, not re-created**: their UUIDs are unchanged, so the PCB footprints stay linked and keep their current placement. No parts were added or removed, and part values are unchanged.

Checks after the change:
- ERC: 0 errors.
- The pre-existing `multiple_net_names` warnings (vout3/vout4) and the 16 `similar_labels` warnings are gone, because the old `out_a2`/`in-_b2`/`vout3`… global labels no longer exist.
- The remaining warnings are the pre-existing library/footprint links (J3, OPA4350) and `S8`.

## New section assignment

Chosen from the PCB pad positions: J3 S1…S7 run top→bottom at x = 125.4, U6 sits above J3 and U13 below. With this mapping the sensor traces can be routed on F.Cu without crossing.

| Package | A (pins 1-2-3) | B (7-6-5) | C (10-11-12) | D (16-15-14) |
|---|---|---|---|---|
| **U6** (above J3) | charge amp **S1** → Q1 | charge amp **S2** → Q2 | charge amp **S3** → Q3 | charge amp **S4** → Q4 |
| **U13** (below J3) | charge amp **S7** → Q7 | charge amp **S_sac** → Vsac | charge amp **S5** → Q5 | charge amp **S6** → Q6 |
| **U11** | diff CH1 → Vout1 | diff CH2 → Vout2 | diff CH3 → Vout3 | diff CH4 → Vout4 |
| **U12** | diff CH5 → Vout5 | diff CH6 → Vout6 | diff CH7 → Vout7 | **spare: follower** (+IN_D = GND, −IN_D = OUT_D = `SPARE_D`) |

In every charge-amp section, +IN goes to GND. Routing intent:
- **U6:**
  - S1/S2 come up the left side (S1 is the outer route, to pin 2).
  - S3/S4 pass under the chip to the right side (S3 is the inner route, to pin 11; S4 the outer, to pin 15).
- **U13:**
  - S7 goes straight to pin 2.
  - S6/S5 pass over the chip to the right side (S6 is the inner route, to pin 15; S5 the outer, to pin 11).
  - S_sac comes from J5, below, to pin 6.

## Passive → channel mapping (refs kept)

| Channel | Charge amp Rf / Cf | Diff amp Rin (Qn→DnN) / Rfb (DnN→Vout) / Rp (Vsac→DnP) / Rg (DnP→GND) |
|---|---|---|
| CH1 (S1) | R22 / C10 | R23 / R24 / R25 / R26 |
| CH2 (S2) | R27 / C12 | R28 / R29 / R30 / R31 |
| CH3 (S3) | R37 / C14 | R40 / R39 / R35 / R36 |
| CH4 (S4) | R32 / C13 | R41 / R38 / R33 / R34 |
| CH5 (S5) | R47 / C16 | R50 / R49 / R45 / R46 |
| CH6 (S6) | R42 / C15 | R51 / R48 / R43 / R44 |
| CH7 (S7) | R57 / C18 | R60 / R59 / R55 / R56 |
| Reference (S_sac) | R10 / C8 | — (Vsac feeds all Rp) |

Decoupling stays with its package: U6 C22/C11, U13 C27/C28, U11 C23/C24, U12 C25/C26 (+2V5 / −2V5).

## What changed in the drawing

- The four "SENSORS x-y" boxes were replaced by two: **CHARGE AMPLIFIERS (U6, U13)** and **DIFFERENCE AMPLIFIERS (U11, U12)**.
- Every op-amp pin now has a short stub with a net label; each channel's network is drawn as its own small circuit next to its op-amp.
- Nets were renamed:
  - charge-amp outputs `out_a1`/`out_b1`/`out_a2`… → `Q1…Q7`;
  - diff-amp inputs `in-_b1`… / `Net-(U6-+IN_B)`… → `D1N…D7N` / `D1P…D7P`.
- Paper size is A3, matching FOGLIO1.
- Behaviour per channel is unchanged: the charge amp is still 1 MΩ ∥ 15 pF, and the diff amp still gives Vout = 2·(Vsac − Qn). Gains are **deliberately untouched**. The first stage was later confirmed as an intentional transimpedance design.

## PCB impact (when you run Update PCB from Schematic)

- No footprints are added or removed. U6/U11/U12/U13 keep their positions.
- **Almost every net in the amplifier area changes pads:**
  - S3/S4 now go to U6, and S5/S6 to U13;
  - the diff-amp resistors now connect to U11/U12 pins;
  - the Rf/Cf of S3–S6 must move next to U6/U13.
- Expect to rip up and re-route that area. The placement notes are on the sheet:
  - Rf/Cf within about 2 mm of the pins (in every section −IN and OUT are adjacent pads);
  - sensor traces on F.Cu only;
  - a GND guard around each −IN;
  - all 8 channels laid out identically.

## Net diff (before → after)

- `/POWER SUPPLY - ADC - SACRIFIED SENSOR/Vsac`
  - before: —
  - after:  (11): C8.2 R10.2 R25.1 R30.1 R33.1 R35.1 R43.1 R45.1 R55.1 R75.1 U13.7
- `/SENSORS/D1N`
  - before: —
  - after:  (3): R23.2 R24.1 U11.2
- `/SENSORS/D1P`
  - before: —
  - after:  (3): R25.2 R26.1 U11.3
- `/SENSORS/D2N`
  - before: —
  - after:  (3): R28.2 R29.1 U11.6
- `/SENSORS/D2P`
  - before: —
  - after:  (3): R30.2 R31.1 U11.5
- `/SENSORS/D3N`
  - before: —
  - after:  (3): R39.1 R40.2 U11.11
- `/SENSORS/D3P`
  - before: —
  - after:  (3): R35.2 R36.1 U11.12
- `/SENSORS/D4N`
  - before: —
  - after:  (3): R38.1 R41.2 U11.15
- `/SENSORS/D4P`
  - before: —
  - after:  (3): R33.2 R34.1 U11.14
- `/SENSORS/D5N`
  - before: —
  - after:  (3): R49.1 R50.2 U12.2
- `/SENSORS/D5P`
  - before: —
  - after:  (3): R45.2 R46.1 U12.3
- `/SENSORS/D6N`
  - before: —
  - after:  (3): R48.1 R51.2 U12.6
- `/SENSORS/D6P`
  - before: —
  - after:  (3): R43.2 R44.1 U12.5
- `/SENSORS/D7N`
  - before: —
  - after:  (3): R59.1 R60.2 U12.11
- `/SENSORS/D7P`
  - before: —
  - after:  (3): R55.2 R56.1 U12.12
- `/SENSORS/Q1`
  - before: —
  - after:  (4): C10.2 R22.2 R23.1 U6.1
- `/SENSORS/Q2`
  - before: —
  - after:  (4): C12.2 R27.2 R28.1 U6.7
- `/SENSORS/Q3`
  - before: —
  - after:  (4): C14.2 R37.2 R40.1 U6.10
- `/SENSORS/Q4`
  - before: —
  - after:  (4): C13.2 R32.2 R41.1 U6.16
- `/SENSORS/Q5`
  - before: —
  - after:  (4): C16.2 R47.2 R50.1 U13.10
- `/SENSORS/Q6`
  - before: —
  - after:  (4): C15.2 R42.2 R51.1 U13.16
- `/SENSORS/Q7`
  - before: —
  - after:  (4): C18.2 R57.2 R60.1 U13.1
- `/SENSORS/SPARE_D`
  - before: —
  - after:  (2): U12.15 U12.16
- `/SENSORS/Vsac`
  - before: (11): C8.1 R10.1 R25.2 R30.2 R33.2 R35.2 R43.2 R45.2 R55.2 R75.1 U13.10
  - after:  —
- `/SENSORS/in-_b1`
  - before: (3): R23.1 R24.1 U6.6
  - after:  —
- `/SENSORS/in-_b3`
  - before: (3): R49.1 R50.2 U12.6
  - after:  —
- `/SENSORS/in-_b4`
  - before: (3): R59.1 R60.1 U13.6
  - after:  —
- `/SENSORS/in-_c1`
  - before: (3): R28.1 R29.1 U6.15
  - after:  —
- `/SENSORS/in-_c3`
  - before: (3): R48.1 R51.1 U12.15
  - after:  —
- `/SENSORS/out_a1`
  - before: (4): C10.1 R22.1 R23.2 U6.1
  - after:  —
- `/SENSORS/out_a3`
  - before: (4): C16.1 R47.1 R50.1 U12.1
  - after:  —
- `/SENSORS/out_a4`
  - before: (4): C18.1 R57.1 R60.2 U13.1
  - after:  —
- `/SENSORS/out_b1`
  - before: (4): C12.1 R27.1 R28.2 U6.10
  - after:  —
- `/SENSORS/out_b3`
  - before: (4): C15.1 R42.1 R51.2 U12.10
  - after:  —
- `GND`
  - before: (95): C11.1 C17.2 C19.1 C20.2 C21.2 C22.2 C23.2 C24.1 C25.2 C26.1 C27.2 C28.1 C33.2 C34.2 C35.2 C36.2 C37.2 C48.2 C49.2 C50.2 C51.2 C52.2 C53.2 C54.2 C55.2 C58.2 C59.2 C60.2 C61.2 C7.1 C9.2 D2.1 H1.1 H2.1 H3.1 H4.1 J2.A12 J2.B12 J2.SH J3.1 J3.10 J3.Z1 J3.Z2 J5.1 J7.1 J7.11 J7.12 J7.3 J7.6 R26.1 R3.2 R31.1 R34.1 R36.1 R4.2 R44.1 R46.1 R56.1 R6.2 R62.2 R64.2 R66.2 R68.2 R70.2 R72.2 R74.2 R76.2 R79.2 R9.2 TP5.1 U1.3 U11.12 U11.3 U12.12 U12.3 U13.12 U13.3 U2.10 U2.11 U2.13 U2.2 U2.25 U2.27 U2.28 U2.3 U2.30 U2.31 U2.6 U2.7 U3.3 U4.13 U4.4 U6.12 U6.3 U7.2
  - after:  (96): C11.1 C17.2 C19.1 C20.2 C21.2 C22.2 C23.2 C24.1 C25.2 C26.1 C27.2 C28.1 C33.2 C34.2 C35.2 C36.2 C37.2 C48.2 C49.2 C50.2 C51.2 C52.2 C53.2 C54.2 C55.2 C58.2 C59.2 C60.2 C61.2 C7.1 C9.2 D2.1 H1.1 H2.1 H3.1 H4.1 J2.A12 J2.B12 J2.SH J3.1 J3.10 J3.Z1 J3.Z2 J5.1 J7.1 J7.11 J7.12 J7.3 J7.6 R26.2 R3.2 R31.2 R34.2 R36.2 R4.2 R44.2 R46.2 R56.2 R6.2 R62.2 R64.2 R66.2 R68.2 R70.2 R72.2 R74.2 R76.2 R79.2 R9.2 TP5.1 U1.3 U12.14 U13.12 U13.14 U13.3 U13.5 U2.10 U2.11 U2.13 U2.2 U2.25 U2.27 U2.28 U2.3 U2.30 U2.31 U2.6 U2.7 U3.3 U4.13 U4.4 U6.12 U6.14 U6.3 U6.5 U7.2
- `Net-(U11-+IN_B)`
  - before: (3): R35.1 R36.2 U11.5
  - after:  —
- `Net-(U11-+IN_D)`
  - before: (3): R33.1 R34.2 U11.14
  - after:  —
- `Net-(U12-+IN_B)`
  - before: (3): R45.1 R46.2 U12.5
  - after:  —
- `Net-(U12-+IN_D)`
  - before: (3): R43.1 R44.2 U12.14
  - after:  —
- `Net-(U13-+IN_B)`
  - before: (3): R55.1 R56.2 U13.5
  - after:  —
- `Net-(U6-+IN_B)`
  - before: (3): R25.1 R26.2 U6.5
  - after:  —
- `Net-(U6-+IN_D)`
  - before: (3): R30.1 R31.2 U6.14
  - after:  —
- `S1`
  - before: (4): C10.2 J3.2 R22.2 U6.2
  - after:  (4): C10.1 J3.2 R22.1 U6.2
- `S2`
  - before: (4): C12.2 J3.3 R27.2 U6.11
  - after:  (4): C12.1 J3.3 R27.1 U6.6
- `S3`
  - before: (4): C14.2 J3.4 R37.2 U11.2
  - after:  (4): C14.1 J3.4 R37.1 U6.11
- `S4`
  - before: (4): C13.2 J3.5 R32.2 U11.11
  - after:  (4): C13.1 J3.5 R32.1 U6.15
- `S5`
  - before: (4): C16.2 J3.6 R47.2 U12.2
  - after:  (4): C16.1 J3.6 R47.1 U13.11
- `S6`
  - before: (4): C15.2 J3.7 R42.2 U12.11
  - after:  (4): C15.1 J3.7 R42.1 U13.15
- `S7`
  - before: (4): C18.2 J3.8 R57.2 U13.2
  - after:  (4): C18.1 J3.8 R57.1 U13.2
- `S_sac`
  - before: (4): C8.2 J5.2 R10.2 U13.11
  - after:  (4): C8.1 J5.2 R10.1 U13.6
- `Vout1`
  - before: (3): R24.2 R61.1 U6.7
  - after:  (3): R24.2 R61.1 U11.1
- `Vout2`
  - before: (3): R29.2 R63.1 U6.16
  - after:  (3): R29.2 R63.1 U11.7
- `Vout3`
  - before: (3): R39.2 R65.1 U11.7
  - after:  (3): R39.2 R65.1 U11.10
- `Vout5`
  - before: (3): R49.2 R69.1 U12.7
  - after:  (3): R49.2 R69.1 U12.1
- `Vout6`
  - before: (3): R48.2 R71.1 U12.16
  - after:  (3): R48.2 R71.1 U12.7
- `Vout7`
  - before: (3): R59.2 R73.1 U13.7
  - after:  (3): R59.2 R73.1 U12.10
- `in-_b2`
  - before: (3): R39.1 R40.1 U11.6
  - after:  —
- `in-_c2`
  - before: (3): R38.1 R41.1 U11.15
  - after:  —
- `out_a2`
  - before: (4): C14.1 R37.1 R40.2 U11.1
  - after:  —
- `out_b2`
  - before: (4): C13.1 R32.1 R41.2 U11.10
  - after:  —
- `unconnected-(U13-+IN_D-Pad14)`
  - before: (1): U13.14
  - after:  —
- `unconnected-(U13--IN_D-Pad15)`
  - before: (1): U13.15
  - after:  —
- `unconnected-(U13-OUT_D-Pad16)`
  - before: (1): U13.16
  - after:  —


---

# Part 3: multi-unit op-amp symbols (FOGLIO2 / SENSORS), readability only

**No electrical change.** The exported netlist is identical to Part 2, net names included. Local labels `D1N…D7N`, `D1P…D7P` (difference-amp − and + inputs) and `SPARE_D` (spare follower loop) are placed on the wires.

## Symbol

- The custom single-box `OPA4350EA_250:OPA4350EA_250` was replaced by KiCad's standard **`Amplifier_Operational:OPA4340EA`**, embedded (flattened from `LTC6082xGN`, which it extends).
  - It has 5 units: A, B, C, D and E (power).
  - Its SSOP-16 pinout is identical to OPA4350EA: 1 OUT A, 2 −A, 3 +A, 4 V+, 5 +B, 6 −B, 7 OUT B, 8/9 NC, 10 OUT C, 11 −C, 12 +C, 13 V−, 14 +D, 15 −D, 16 OUT D.
  - The NC pins 8/9 are "no-connect" pins inside unit A, so no flags are needed.
- **Kept from your symbol:** Value `OPA4350EA_250`, Footprint `Opa4:SOP63P600X175-16N`, and all SnapEDA/purchasing fields (MF, MPN, MP, OC_FARNELL, OC_NEWARK, Purchase-URL, …). Datasheet set to TI's OPA4350.
- **PCB link:** unit A of each package reuses the old symbol UUID. The netlist `tstamps` for U6/U11/U12/U13 contain the UUID the PCB footprints point to, so the footprints stay linked and in place.

## Drawing

- Every channel is now a conventional op-amp circuit: − input on top, Rf ∥ Cf (or Rfb) drawn as the feedback loop, + input to GND (charge amps) or to the Rp/Rg divider (diff amps).
- Charge amps: left column U6 A–D (S1–S4), right column U13 C, D, A, B (S5, S6, S7, S_sac).
- Diff amps: U11 A–D (CH1–4) and U12 A–C (CH5–7). U12 D is the spare follower.
- Each package's power unit (E) sits at the bottom of its column, next to its two decoupling caps.
- All passives keep their references, values and UUIDs; only positions changed.

## ERC

- 0 errors.
- `lib_symbol_issues` 5 → 1: only J3's custom library is left.
- `footprint_link_issues` 5 → 21: this is the **same pre-existing warning** ("footprint library `Opa4` not in the configuration"), now counted once per unit (4 packages × 5 units + J3). It goes away once you add the `Opa4` footprint library to the project's footprint-library table (or switch to `Package_SO:SSOP-16_3.9x4.9mm_P0.635mm`).


---

# Part 4: USB input TVS + pi filter (FOGLIO1)

Designed with Murata SimSurfing for the 0.15–10 MHz band, C + bead + C topology.

| Change | Ref | Value / part | Footprint | Pins → nets | Position |
|---|---|---|---|---|---|
| **Added** | C62 | 22u 16V, `ZRB18AR61C226ME01L` (Murata, X5R, low acoustic noise) | `Capacitor_SMD:C_0603_1608Metric` | 1 = +5V, 2 = GND | (30.48, 99.06) |
| **Added** | FB2 | `BLM18KG102SN1D` (1000 Ω @ 100 MHz, 1 A, Rdc ≤ 0.2 Ω) | `Inductor_SMD:L_0603_1608Metric` | 1 = +5V, 2 = **+5V_CP** | (38.1, 96.52) |
| **Removed** | C21 | 10u 0805 | – | was +5V / GND | (54.61, 59.69) |
| **Modified** | #PWR023 | The +5V power symbol on U4's input network was replaced by the local label `+5V_CP` at the same point (59.69, 46.99) | – | – | – |
| **Added** | #FLG06 | PWR_FLAG on `+5V_CP` (it is fed through a passive bead) | – | – | (45.72, 96.52) |
| **Added** | D1 | **SMF5.0A** (Littelfuse), 5 V stand-off unidirectional TVS, 200 W (10/1000 µs), V_BR ≥ 6.4 V, V_C ≤ 9.2 V @ 21.7 A. KiCad symbol `Diode:SMF5V0A` (embedded, flattened) | `Diode_SMD:D_SMF` | 1 (cathode) = +5V, 2 (anode) = GND | (22.86, 100.33) |

Resulting nets (from the exported netlist):
- `+5V` (connector side of FB2): J2.A9/B9, D1 (TVS), R7 (LED), TP1, U7.IN/EN, C60, C62.1, FB2.1
- `+5V_CP` (charge-pump side): FB2.2, U4.3 VIN, C34 (10u), C33 (100n), R11 (PGOOD pull-up), R12 (EN pull-up)

Notes:
- The U4-side capacitor of the pi filter is the existing C34 (+ C33), placed right at U4 as its datasheet requires. C21 was dropped to reduce the bulk capacitance on VBUS (now about 22 µF + 10 µF + 1 µF nominal).
- U7 (ADC LDO) and the LED are deliberately on the connector side, so U4's 2 MHz input ripple is blocked by FB2 + C62 before it reaches them.
- Layout: D1 and C62 right at J2's VBUS/GND pins (D1 first, short fat GND via), FB2 between J2 and U4, C34/C33 tight to U4 VIN/GND (unchanged).
- D1 choice: over TI TVS0500 because the SMF starts conducting ~1.5 V earlier (6.4 V vs 7.5 V min), which is what matters for low-current hot-plug ringing; same ~9.2 V clamp at full surge; its 400 µA max leakage is irrelevant on a ~200 mA USB rail; hand-solderable. Neither TVS holds VBUS below the LM27762's 5.8 V abs-max; an OVP load switch (e.g. TPD1S514 family) would be the fix for a faulty charger (not added).
- ERC: 0 errors (same warnings as before).
- **Update PCB from Schematic** will remove C21's footprint, add D1, C62 and FB2, and move U4.3/C34/C33/R11/R12 onto the new net `+5V_CP`.

---

# Part 5: part numbers, value and part changes for procurement

Checked on Mouser Italy on 2026-09-28. The ordering file is **`PCB/bom/BOM_Mouser_2026-09-28.csv`**, with quantities, MPNs, Mouser links, stock and EUR prices at qty 10 and 100.

- **MPN + Manufacturer fields** added to all 123 parts that have a part to order. The old SnapEDA `MF` field was replaced by `Manufacturer`.
- **Diff-amp 5k → 4.99k** (R23, R25, R28, R30, R33, R35, R40, R41, R43, R45, R50, R51, R55, R60). Exact 5.0 kΩ only exists at 0.1 %/25 ppm as about a €1.7 part. The same 4.99k/10k pair is used on both sides of every diff amp, so the ratio stays matched (gain 2.004).
- **U7 TPS7A2033PDBVR → LP5907MFX-3.3/NOPB.** The TPS7A2033 was out of stock with a 26-week lead. The LP5907 has the same SOT-23-5 pinout (IN 1, GND 2, EN 3, NC 4, OUT 5) and the same footprint. Symbol is now `Regulator_Linear:LP5907MFX-3.3`, and the netlist is unchanged. Note its VIN max is 5.5 V vs 6.0 V.
- **C62 → Murata GRM219R61C226ME15K, 22 µF 16 V X5R, footprint changed to 0805** (the ZRB 0603 was out of stock until mid-October).
- **Capacitor MPNs:** 15 pF C0G = Samsung CL10C150JB8NNNC (every Murata 0603 15 pF C0G is NRND). 100 nF = GRM188R72A104KA35D, 1 µF = GRT188R71E105KE13D and 4.7 µF = GRT188R61C475KE13D, replacing obsolete or unorderable parts. 10 µF 0805 = GRJ21BR61E106KE01L.
- **Precision resistors:** 1 MΩ and 1 kΩ are KOA RN73R (0.1 %, 25 ppm); 4.99k, 10k and 1.1k are Yageo RT0603BRD. The general-purpose parts are Yageo RC0603FR-07, except 105k = AC0603FR-07105KL.
- ERC: 0 errors. The netlist is identical to Part 4.
- **Open points:**
  - J5 has no MPN yet: it needs a 1x03 1.27 mm SMD header with alternating pins to match the `_SMD_Pin1Right` footprint (changed from 2.54 mm to 1.27 mm pitch, see Part 6), plus a 1.27 mm shunt if it is used as a jumper.
  - OPA4350EA/250 is **end of life** (TI Last Time Buy): buy the needed quantity now.
  - Mouser stock not yet checked for D2, J2, J3, J7 and U1.
- **Update PCB from Schematic** changes C62's footprint to 0805 (re-place it at J2) and updates U7's value.

---

# Part 6: J5 to 1.27 mm pitch (FOGLIO1)

- J5 (Conn_01x03_Pin: pin 1 GND, pin 2 S_sac, pin 3 S8) footprint:
  `Connector_PinHeader_2.54mm:PinHeader_1x03_P2.54mm_Vertical_SMD_Pin1Right`
  → `Connector_PinHeader_1.27mm:PinHeader_1x03_P1.27mm_Vertical_SMD_Pin1Right`.
- Same pin-1-right alternating layout, so the pin sides don't change: pins 1 and 3 on the right, pin 2 on the left.
- Pads: 3.0 × 0.65 mm at x ±1.5, y −1.27 / 0 / +1.27. Courtyard 7.0 × 4.82 mm, where the 2.54 mm version was about 5.1 mm tall between pad centres.
- Netlist: the only change is J5's footprint field; every net is identical.
- ERC: 0 errors; the same 31 warnings as before (library-config and C_Small symbol-copy warnings).
- The BOM row for J5 is updated to search for 1.27 mm headers; the MPN is still to be chosen.
