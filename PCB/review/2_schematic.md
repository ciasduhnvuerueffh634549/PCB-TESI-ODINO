# Review #2: Schematic electrical correctness

**Scope.** `PCB v2.kicad_sch`, `FOGLIO1.kicad_sch` and `FOGLIO2.kicad_sch` at HEAD `c884b04` (working tree clean). Every check ran on a copy in `scratchpad/review/schematic2/` using kicad-cli 10.0.6.

**Datasheets used** (downloaded from ti.com and read with pdftotext):
- ADS131M08: SBAS950B
- LM27762: SNVSAF7C
- LP5907
- OPAx350: SBOS099D
- SN74LVC1G17
- TPD2E2U06

**Method.**
1. ERC.
2. `kicad-cli sch export netlist`, parsed pin by pin.
3. Every schematic pin compared against the PCB pad net, parsed from `.kicad_pcb`.
4. Every IC pinout compared against the datasheet pin table.
5. BOM CSV compared against the netlist (MPN, value, footprint, manufacturer and quantity).

## Headline

**No BLOCKER found.** Three facts support this:
- **Schematic and PCB agree.** Comparing netlist to PCB pads showed **zero** net, footprint, value or MPN differences, for all 132 footprints and every pad.
- **Every IC pinout matches its datasheet.** U2, U3, U4, U6/U11/U12/U13, U7 and U1 were checked pin by pin. Their footprints have the right pad count, numbering and exposed-pad treatment.
- **BOM CSV agrees with the schematic.** MPN, manufacturer and quantity match for every row. The only mismatch is the already-known R77–R79 footprint text.

There is one new SHOULD-FIX (the 10 k precision/tolerance ambiguity) and a set of NOTEs.

---

## 1. ERC triage

Result: 0 errors, 31 warnings. Four checks are ignored in the project: global label only once, four-way junction, SPICE model, and footprint filter.

| Class | Count | Verdict |
|---|---|---|
| `footprint_link_issues`: `Opa4` library (U6/U11/U12/U13, all units) and `CONNETTORE 687110182122` (J3) not in a library table | 21 | Benign. The footprints are embedded in the board, and the netlist-to-PCB comparison is clean. Only "update from library" is affected. (known, DESIGN §9.1 item 12) |
| `lib_symbol_issues`: J3 symbol library `6871XX182122_687110182122` not in a library table | 1 | Benign. The cached symbol is used. |
| `lib_symbol_mismatch`: `C_Small` differs from `Device` (C11, C22–C28) | 8 | Benign. It is a local symbol copy; the pins match (netlist verified). |
| `same_local_global_label` `S8` | 1 | Benign. See the detail after this table. |

**`S8` detail.**
- The root sheet has **one** global label `S8`, at (40.64, 67.31). That is exactly the position of the FOGLIO1 sheet pin `S8`. Its single appearance is masked by the ignored "global label only once" check.
- FOGLIO1 has a local `S8` plus the hierarchical `S8`.
- The resulting net `S8` has exactly J3.9 and J5.3, which is correct.
- The root global label is a dead stub. It can be deleted, and is harmless as it is.

**Other checks that passed.**
- **Single-pin nets:** only the 11 intentional no-connects. These are U2.22 XTAL2, U3.1, U7.4, and pins 8/9 on each of the four OPA4350s.
- **Dangling labels:** none.
- **Labels differing only by case:** none.
- **Hierarchy:** hierarchical labels and sheet pins match (16 on SENSORS, 17 on POWER). The sheet-pin *directions* for Vout1–7 on the SENSORS sheet say `input` although they are outputs. This is cosmetic only.
- **Power flags:** 3 `PWR_FLAG`s, and there are no "power input not driven" errors.

**Hidden pins.** They exist only where they are stacked and correctly joined:
- U2 pins 27/28 are stacked on AGND 13.
- J2 B9/B12 are stacked on A9/A12.
- The NC pins (U3.1, U7.4, OPA pins 8/9) are hidden no-connects.

No hidden pin ties a supply by name.

---

## 2. Findings

### SHOULD-FIX

**S-1. The 0.1 % diff-amp resistors are distinguished only by MPN; their Value is plain "10k".**
- **Location.** R24, R26, R29, R31, R34, R36, R38, R39, R44, R46, R48, R49, R56, R59 are `10k`, RT0603BRD0710KL (0.1 %, 25 ppm). R77, R78, R11 and R12 are also `10k`, but RC0603FR-0710KL (1 %).
- **Evidence.** A fresh `kicad-cli sch export bom --group-by Value,Footprint` merges R24…R59 with R77/R78 into **one line with two MPNs**: `"R24,…,R77,R78","10k",…,"RC0603FR-0710KL,RT0603BRD0710KL"`. The hand-made Mouser CSV separates them correctly, but DESIGN §7 item 10 says all outputs will be regenerated.
- **Impact.** An assembler or a regenerated BOM could fit 1 % parts in the diff amps. Worst-case resistor-limited CMRR falls from about 57 dB to about 37 dB, i.e. (1+G)/(4·0.01) ≈ 75. The Vsac subtraction, the core of the thesis, then degrades.
  - 4.99 k, 1.1 k, 1 k and 1 M are not affected, because each exists only as a precision part.
- **Fix** (schematic-only, zero layout impact). Either:
  - set Value to `10k 0.1%` on the 14 diff-amp resistors (and, for consistency, on the 4.99 k / 1 M / 1.1 k / 1 k parts); or
  - add a `Tolerance` field and group the BOM by MPN.

### NOTE (no change needed before ordering)

**N-1. ADS131M08 SCLK and DIN float when J7 is unplugged.**
- **Location.** U2.19 (ADC_SCLK) and U2.21 (ADC_MOSI) have no pull resistor. CS is held high by R77 and SYNC/RESET by R78.
- **Evidence.** SBAS950B §9.1.1: "Do not float unused digital inputs because excessive power-supply leakage current can result."
- **Impact.** During the bring-up steps with J7 unplugged (DESIGN §9.2 step 2), DVDD current may be higher than expected. There is no damage and no functional effect, because CS is high.
- **Fix.** Next revision: 100 k pull-downs on SCLK and MOSI. For now, do not read anything into the +3V3 current until J7 is connected.

**N-2. J7 back-power path into U2** (known, DESIGN §9.2 step 5).
- **Location.** ADC_SCLK, ADC_MOSI, ADC_CS and ADC_SYNC_RESET go straight from J7 to U2 pins 19, 21, 17 and 16.
- **Evidence.**
  - SBAS950B §6.1 digital input maximum is DVDD + 0.3 V, with **±10 mA continuous input current**.
  - An H7 push-pull GPIO driving into an unpowered +3V3 rail through U2's clamp diodes is limited only by the GPIO. The 33 Ω recommended at the H7 end does not limit it.
  - R77/R78 also feed about 0.33 mA each from a driven-high CS/SYNC line into +3V3.
  - U3 (LVC, Ioff-protected) is safe. U2's four direct inputs are not.
- **Impact.** Stress or latch-up risk if the H7 is powered first.
- **Fix.** Power the board before the H7 (already in DESIGN). If the order cannot be guaranteed, use about 330 Ω–1 kΩ series resistors at the H7 end instead of 33 Ω: at 8 MHz with about 10 pF, the RC is 3–10 ns, which is acceptable.

**N-3. D1 symbol graphic does not show polarity.**
- **Location.** D1 `Diode:SMF5V0A`, pins named `A1` (to +5V) and `A2` (to GND), which is a bidirectional-TVS style.
- **Evidence.**
  - The part SMF5.0A is **unidirectional**.
  - Footprint `D_SMF` pad 1 is on +5V. The closed silk end is at x = −2.36, the pad-1 side, i.e. the cathode.
  - The Description field says "Pin 1 = cathode".
- **Impact.** None on the board: the cathode goes to VBUS, which is correct. The schematic simply doesn't show which end is the cathode.
- **Fix** (optional). Swap to a unidirectional TVS symbol with K/A pins.

**N-4. LM27762 PGOOD is pulled up instead of grounded.**
- **Location.** U4.1 → R11 10 k → +5V_CP.
- **Evidence.** SNVSAF7C Table 4-1: "Connect to ground if not used". Table 7-1 lists R_PU as optional.
- **Impact.** Valid as drawn. It costs about 0.5 mA continuously once power is good (open drain, logic 0 = good). (known, DESIGN §9.1 item 14)

**N-5. REFIN has a 100 nF capacitor (C59).**
- **Evidence.**
  - This matches the characterisation condition in SBAS950B §6.5.
  - §8.3.4 and the Table 5-1 note (2) say: "Do not place any capacitance on REFIN if the current-detect mode is used."
- **Action** (firmware). Never enable current-detect mode.

**N-6. AIN versus AVDD during power sequencing.**
- **Evidence.**
  - SBAS950B §10.2 says inputs must never exceed the supply limits.
  - The absolute range is AGND − 1.6 V to AVDD + 0.3 V, at ±10 mA.
- **Analysis.** If ±2V5 is up while +3.3VA is not (unlikely: U7 starts from raw +5V), an AIN can reach +1.17 V with AVDD = 0. The current is then limited by the 1.1 kΩ top resistor to under 1.5 mA, well within 10 mA.
- **Conclusion.** No action.

**N-7. Symbol and library hygiene** (cosmetic).
- The op-amps use a symbol named `OPA4340EA` for OPA4350 parts. The pinout is identical to the OPA4350 SSOP-16 (verified).
- FOGLIO1's `lib_symbols` still caches an unused `AD7606` symbol.
- The root global `S8` is a stub (§1).

**N-8. J5 in position 1–2 leaves the S8 electrode floating.** This is expected (the reference is grounded), but the 8th flex electrode then sits unterminated next to S7. Mention it in the bring-up notes; no hardware change.

**N-9. Sensor-input protection** (known, DESIGN §9.1 item 7).
- **Evidence.** OPA4350 signal inputs are limited to (V−) − 0.3 V … (V+) + 0.3 V and 10 mA (SBOS099D §6.1). S1–S8 reach −IN with only Rf ∥ Cf.
- **Impact.** An impact spike that saturates stage 1 can forward-bias the input clamps, including when the board is unpowered. Next revision: add series R and/or clamps.

**N-10. VBUS margins** (known, DESIGN §9.1 item 8).
- LP5907 VIN operating maximum is 5.5 V and absolute maximum 6 V.
- LM27762 absolute maximum is 5.8 V.
- SMF5.0A clamps at up to 9.2 V.

Lab use only.

**N-11. Series-part identity.** OPA4350EA/250 is Last-Time-Buy; /2K5 is Active (confirmed in the SBOS099D orderable addendum). (known)

**N-12. BOM hygiene** (known).
- R77–R79 footprint text: the CSV says `PCM_SparkFun-Resistor`, the schematic says `Resistor_SMD`.
- TP1/2/3/5 are `in_bom yes` but absent from the CSV. H1–H4 are also absent, correctly.
- There is no mating FFC or J7 cable in the BOM.

### UNVERIFIED (information not in the repo)

**U-1. J3 contact side and flex tail.**
- The repo has no drawing of the sensor flex.
- The netlist has J3.1 = GND, J3.2–9 = S1–S8, J3.10 = GND, and Z1/Z2 = GND.
- Because GND sits on **both** end pins, a mirrored or wrong-side flex only permutes the channel order (S1↔S8, S2↔S7, …). Nothing is shorted to a rail, so the failure is recoverable in firmware or by moving J5.
- **To confirm:** the flex drawing (pin-1 side, contact face) against the Würth 687110182122 datasheet (top or bottom contact).

**U-2. J7 against the STM32H7 host.**
- No host board or pin map is identified in the repo.
- The J7 signals and levels are consistent with any 3.3 V H7 SPI plus MCO2. MCO2 on PC9 is itself unverified in DESIGN.
- **To confirm:** check the chosen H7 board's pins for MCO2, SPIx SCK/MOSI/MISO, and a GPIO or EXTI for DRDY.

**U-3. D2 brightness.** The current is (5 − V_f) / 4.7 k, about 0.4–0.6 mA depending on the unverified V_f of Würth 150060GS75000. The LED may be dim. (known, DESIGN §9.1 item 16)

---

## 3. Per-IC checklist

| IC | Checked | Result |
|---|---|---|
| **U2 ADS131M08IPBSR** (TQFP-32 PBS). SBAS950B Table 5-1, §6.1, §6.3, §8.3.4, §8.3.5, §9.1.1, §10.1, §10.3 | Pins 1–32 against Table 5-1; the footprint pad map is identical | **Pass** |
| | Supplies: AVDD 15 = +3.3VA with C37 1 µF; DVDD 26 = +3V3 with C36 1 µF (§10.3: 1 µF each). AGND 13/28 and DGND 25 go to the single GND. | Pass |
| | Pin 27 (NC per datasheet; the symbol calls it "AGND") is tied to GND. The datasheet allows "leave unconnected or connect to AGND". | Pass, with remark |
| | CAP 24 → C35 220 nF, 25 V to GND (§10.1: 220 nF when DVDD > 2.7 V) | Pass |
| | REFIN 14 → C59 100 nF; internal 1.2 V reference (§8.3.4) | Pass; see N-5 |
| | AIN0N–AIN7N to GND. AINxP from 1.1 k / 1 k + 10 nF C0G. ±1.165 V lies within the ±1.2 V full scale and the AGND − 1.3 V absolute input range (§6.3). | Pass |
| | CLKIN 23 ← R80 33 Ω ← U3. 5.12 MHz lies within 0.3–8.4 MHz (§6.3). XTAL2 22 is not connected. | Pass (firmware: HR mode, 40–60 % duty) |
| | SYNC/RESET 16 has a 10 k pull-up; CS 17 has a 10 k pull-up. DRDY 18 and DOUT 20 have 33 Ω series resistors at U2. SCLK 19 and DIN 21 are driven directly. | Pass; see N-1 and N-2 |
| | No exposed pad on the TQFP (Table 5-1 lists a thermal pad for WQFN only); the footprint has none | Pass |
| **U6, U13, U11, U12 OPA4350EA** (SSOP-16 DBQ). SBOS099D p.3, §6 | Pin map: 1/7/10/16 OUT; 2/6/11/15 −IN; 3/5/12/14 +IN; 4 V+; 13 V−; 8/9 NC. Matches the SSOP column exactly, and the footprint pads match. | **Pass** |
| | Supplies: +2V5 and −2V5 = 5.0 V (2.7–5.5 V operating). A 100 nF decoupler per rail per package (C22/C11, C27/C28, C23/C24, C25/C26). | Pass |
| | Stage 1: +IN = GND; Rf 1 M (0.1 %) ∥ Cf 15 p C0G between −IN and OUT on all 8 sections. The input CM of 0 V is inside −0.1 … V+ + 0.1. | Pass |
| | Diff amps: Rin 4.99 k (Q→−IN), Rf 10 k (−IN→OUT), 4.99 k (Vsac→+IN), 10 k (+IN→GND), all 0.1 % / 25 ppm by MPN. All 7 channels traced; ratio 2.004 on both halves. | Pass; tolerance visible only by MPN (**S-1**) |
| | U12-D spare: follower, +IN at GND | Pass |
| | Loads: Vsac drives about 1.06 kΩ (7 × 14.99 k ∥ 2.1 k); the swing is within 200 mV of the rails into 1 kΩ (§6); I_out ≪ 40 mA | Pass |
| **U4 LM27762DSSR** (WSON-12 DSS). SNVSAF7C Table 4-1, §5.1, §5.5, Eq. 1 and 3, §7.2.2.4 | Pin map 1–12 plus EP pad 13 = GND; EP 2.65 mm per DSS0012B | **Pass** |
| | +2V5 = 1.2 × (130 k + 120 k) / 120 k = 2.500 V; R6 = 120 k ≥ 50 k | Pass |
| | −2V5 = −1.22 × (105 k + 100 k) / 100 k = −2.501 V; R9 = 100 k ≥ 50 k | Pass |
| | Tolerance: VFB± ±1.5 % plus 1 % resistors gives about ±2.5 % → ±2.44…2.56 V; the op-amp span is ≤ 5.12 V < 5.5 V | Pass |
| | C1 flying cap: C6 1 µF 25 V X7R (0.47–1 µF recommended) | Pass |
| | CP: C17 10 µF 25 V (10 µF recommended at high current) | Pass |
| | C_IN: C34 10 µF 25 V + C33 100 nF (4.7 µF recommended; ≥ 2 µF effective) | Pass |
| | C_OUT±: C20/C19 4.7 µF 16 V + 100 nF (≥ 2.2 µF required) | Pass |
| | EN+ and EN− tied, 10 k to VIN (EN max = VIN) | Pass |
| | PGOOD pulled up | Pass; see N-4 |
| | Load: ≤ 120 mA per rail at the OPA4350 maximum I_Q (7.5 mA × 16), below 250 mA | Pass |
| **U7 LP5907MFX-3.3** (SOT-23-5 DBV). Table 4-2, §5.1, §5.6 | 1 IN, 2 GND, 3 EN, 4 N/C, 5 OUT; the symbol and footprint match | **Pass** |
| | EN tied to IN (+5V); EN absolute maximum is 6 V | Pass |
| | C_IN: C60 1 µF | Pass |
| | C_OUT: C61 1 µF + C37 + (via FB1) C36 + C58, about 3.1 µF (0.7–10 µF allowed) | Pass |
| | VIN max 5.5 V | See N-10 (known) |
| **FB1 BLM18PG221SN1D** | +3.3VA → +3V3 (DVDD, U3, pull-ups). Load is a few mA against a rating of about 1.4 A (unverified exact rating; large margin either way). | Pass |
| **FB2 BLM18KG102SN1D** | 1000 Ω, **1 A**, 0.2 Ω maximum (Murata spec via distributor listing) against about 0.2 A | Pass |
| **U3 SN74LVC1G17DBVR** (SOT-23-5). Datasheet p.3 | 1 NC, 2 A, 3 GND, 4 Y, 5 VCC. VCC = +3V3 with C58 100 nF. Input pulled down by R79 100 k; output → R80 33 Ω → CLKIN. | **Pass** |
| **U1 TPD2E2U06DCKR** (SC-70). §5 | 1 IO1 = CC1, 2 IO2 = CC2, 3 GND. V_RWM 5.5 V is fine for CC. | **Pass** |
| **J2 USB4125-GF-A** | A9/B9 VBUS = +5V; A12/B12 and 4 × SH = GND. CC1 (A5) → R4 5.1 k and CC2 (B5) → R3 5.1 k to GND, so a C-to-C cable delivers 5 V. Shield tied directly to GND (acceptable for a lab board). | **Pass** |
| **D1 SMF5.0A** | Cathode (pad 1, silk mark) → +5V. Anode → GND. | Pass; see N-3 |
| **D2 LED** + R7 4.7 k | K (pad 1) → GND, A → R7 → +5V. R7 dissipation is under 3 mW. | Pass; see U-3 |
| **J3** 687110182122 | 1/10/Z1/Z2 = GND; 2–9 = S1–S8. Each S net has exactly 4 nodes (J3, −IN, Rf, Cf); S8 has 2 (J3.9, J5.3). | Pass; see U-1 |
| **J5** M50-3630342 | 1 GND, 2 S_sac, 3 S8. Alternating-tail footprint; either 180° orientation fits. | Pass |
| **J7** 61001221121 | 1 SYNC_RESET, 2 MCU_CLK, 4 MOSI, 5 CS, 8 MISO, 9 DRDY, 10 SCLK; 3/6/7/11/12 GND. Odd/even numbering matches the footprint. No supply pin (by design). | Pass; see U-2 |
| **Test points** | TP1 +5V, TP2 +2V5, TP3 −2V5, TP5 GND (matches DESIGN §6). None for +3.3VA or +3V3. | Pass (known gaps) |

**Passive ratings.** Every capacitor is rated at least 5× its rail, except C62 at 3.2×:
- 100 nF: 100 V.
- 1 µF: 25 V.
- 4.7 µF on ±2.5 V: 16 V.
- 10 µF on 5 V and on −5 V (CP): 25 V.
- 22 µF on VBUS: 16 V (3.2×, adequate for X5R on 5 V).
- C0G parts: 50 V.

C0G is used where it matters: Cf and the anti-alias capacitors. The highest resistor dissipation is about 3 mW (divider at full swing), against a 0.1 W rating for 0603.

**BOM CSV against the schematic** (script comparison).
- All 124 references are present, with the right quantity per row.
- MPN and manufacturer match 100 %.
- Values match.
- Footprint differences: only R77, R78 and R79 (known).
- Only in the schematic, not the CSV: TP1–TP5 (known) and H1–H4 (correct).
- Only in the CSV: the J5 shunt (correct; not a PCB part).
