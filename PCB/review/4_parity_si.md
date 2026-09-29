# Review #4: schematic ↔ PCB parity and signal integrity

Board: `PCB v2.kicad_pcb` at git `c884b04` (branch `riccardo_fix`, clean). All tools were run on a scratch copy (`scratchpad/review/reviewer4/`) with kicad-cli 10.0.6 and the pcbnew Python API. No repo files were touched and no `mcp__kicad__*` tool was used.

**Method**
- Exported the netlist (`kicad-cli sch export netlist`).
- Ran DRC with `--schematic-parity --severity-all` and ran ERC.
- Parsed the board with pcbnew: 626 tracks/arcs, 214 vias, 132 footprints, zone fills.
- Re-filled all zones on the copy. The fills were identical (In1 2410.2 mm², In2 2020.6 mm², B.Cu −2V5 1960.1 mm²), so the saved fills are current.
- Measured coupling with a custom 2-D finite-difference field solver for the cross-section: 0.1 mm × 35 µm trace, 0.1 mm FR-4 (εr 4.5) over a plane, no solder mask.

**Solver results (2-D, no mask; used for every coupling estimate below)**

| Quantity | Value | Notes |
|---|---|---|
| C to plane (C11) | 0.079 pF/mm | DESIGN says 0.10 |
| Z0 | ≈ 68 Ω | about 63 Ω with solder mask (estimate); DESIGN says 59 Ω |
| t_d | ≈ 5.4 ps/mm | about 5.7 with mask |
| Mutual C at 0.2 mm edge gap | 3.3 fF/mm | Cm/C = 0.041; Lm/L = 0.12 |
| Mutual C at 0.5 mm gap | 0.56 fF/mm | |
| Mutual C at 1.0 mm gap | 0.13 fF/mm | |

---

## 1. Parity result

| Check | Schematic | PCB | Mismatches |
|---|---|---|---|
| Footprints / symbols (on_board) | 132 | 132 | **0** missing, **0** extra |
| Value | 132 | 132 | **0** |
| Footprint lib ID | 132 | 132 | **0** |
| Pins / pads (unique ref+pad) | 386 | 386 | **0** pad-net mismatches; the multi-pad U4 EP and the J2 shells are consistent |
| Nets | 88 | 88 (+ the no-net "") | **0** |
| DNP / exclude-from-board | none / none | none | **0** |
| Exclude-from-BOM | H1–H4 | H1–H4 | **0**. TP1/2/3/5 are in_bom in both (known, DESIGN §7.6) |
| `kicad-cli pcb drc --schematic-parity` | | | **0 parity issues, 0 unconnected** |
| BOM CSV vs PCB | 124 refs + J5 shunt | | values 0 diffs; footprint text differs only on R77–R79 (known §7.5) |
| ERC | | | 0 errors, 31 warnings: 21 missing-library, 8 C_Small, 1 label, 1 symbol-lib (known §5.11) |

**Channel map: consistent everywhere.**
- The netlist gives:
  - AIN0P ← R61 ← Vout1;
  - AIN1P ← R63 ← Vout4;
  - AIN2P ← R65 ← Vout2;
  - AIN3P ← R67 ← Vout3;
  - AIN4P ← R69 ← Vout5;
  - AIN5P ← R71 ← Vout6;
  - AIN6P ← R73 ← Vout7;
  - AIN7P ← R75 ← Vsac.
- The U2 pads are 29/32/1/4/5/8/9/12. They match the TI ADS131M08 datasheet (SBAS950B, Table 5-1): AIN0P = 29, AIN1P = 32, AIN2P = 1, AIN3P = 4, AIN4P = 5, AIN5P = 8, AIN6P = 9, AIN7P = 12.
- All AINxN pins (30, 31, 2, 3, 6, 7, 10, 11) are on GND, and pin 27 (NC) is on GND, which the datasheet permits.
- The schematic sheet note (FOGLIO1 text), DESIGN §2.1 and DESIGN §10.3 (`ain_to_sensor[] = {1,4,2,3,5,6,7,0}`) all agree.

**J7 pads vs nets vs silk: all 12 correct.**

| Pin | Net | Silk label |
|---|---|---|
| 1 | ADC_SYNC_RESET | SYNC |
| 3 | GND | GND |
| 5 | ADC_CS | CS |
| 7 | GND | GND |
| 9 | ADC_DRDY | DRDY |
| 11 | GND | GND |
| 2 | MCU_CLK | CLK |
| 4 | ADC_MOSI | MOSI |
| 6 | GND | GND |
| 8 | ADC_MISO | MISO |
| 10 | ADC_SCLK | SCLK |
| 12 | GND | GND |

- Every label sits at its own pad's x: 158.68, 161.22, 163.76, 166.3, 168.84 or 171.38.
- Odd-row labels are above the pads (y ≈ 80.2–80.9); even-row labels are below them (y ≈ 91.9–93.5).

**J5: correct.**

| Pin | Pad position | Net | Silk label |
|---|---|---|---|
| 1 | (124.58, 110.48) | GND | "GND" at (123.38, 108.48) |
| 2 | (125.85, 113.48) | S_sac | "S_SAC" at (124.1, 113.73) |
| 3 | (127.12, 110.48) | S8 | "S8" at (128.12, 108.48) |

**Test points: nets correct.**

| TP | Net | Position | Label |
|---|---|---|---|
| TP1 | +5V | (123.75, 97.90) | "+5V" at (122.78, 96.53) |
| TP2 | +2V5 | (123.53, 94.17) | "+2V5" at (122.70, 92.56) |
| TP3 | −2V5 | (125.01, 83.77) | "−2V5" at (126.36, 84.14) |
| TP5 | GND | (125.01, 87.52) | "GND" at (126.36, 87.89) |

- Minor: the "+5V" label is 1.7 mm from TP1 but only 2.5 mm from TP2, so it reads as sitting between the two pads. Consider nudging it (NOTE).

**Channel silk (B.Silk; mirrored = True on all 15): correct.**
- AIN0…AIN7 each sit within ≤ 0.25 mm of their own R_top:
  - R61 (156.08, 93.81);
  - R63 (159.28, 96.16);
  - R65 (156.08, 98.51);
  - R67 (159.28, 100.86);
  - R69 (156.08, 103.21);
  - R71 (159.28, 105.56);
  - R73 (156.08, 107.91);
  - R75 (159.28, 110.26).
- CH1–CH7 each sit next to their own diff-amp group:
  - CH1 by R26;
  - CH2 by R31;
  - CH3 by R39/R40;
  - CH4 by R41/R33;
  - CH5 by R46;
  - CH6 by R44;
  - CH7 by R55/R59.
- NOTE: CHn follows the sensor/Vout number and AINn follows the ADC channel. CH4 → AIN1, CH2 → AIN2, CH3 → AIN3; the others are off by one. This is correct, but easy to misread during bring-up.

---

## 2. Findings

### BLOCKER
None found in scope. Parity, channel map, J7/J5/TP pinouts and the silk are all consistent. No SI or PI geometry problem would make the boards scrap.

### SHOULD-FIX

**S1. The J7 cable is the weakest SI link, and the board relies entirely on MCU-side measures (partly known: DESIGN §4.2 says "33 Ω at H7").**
- *Location:* J7; IDC ribbon order 1…12.
- *Evidence: ribbon neighbours.*
  - DRDY (pin 9) sits between MISO (8) and SCLK (10). It has **no GND neighbour** and is the MCU's edge-triggered interrupt.
  - CS (5) sits between MOSI (4) and GND (6).
  - CLK (2) sits between SYNC (1) and GND (3).
- *Evidence: termination.*
  - There is no series or RC element on the board at the receiving end of SCLK, MOSI or CS. They go straight into U2's CMOS inputs; the ADS131M08 datasheet specifies no hysteresis.
  - A 33 Ω MCU-side resistor plus an H7 driver of about 20–40 Ω (unverified) is roughly 55–75 Ω. A ribbon is roughly 100–130 Ω (estimate), so the line is under-terminated and edges will overshoot and ring.
- *Impact.*
  - Non-monotonic SCLK edges can double-clock and corrupt frames. CRC will catch this, but at the cost of data loss.
  - SCLK/MISO crosstalk onto DRDY in the ribbon (estimate 3–8 %, i.e. 0.1–0.25 V) can fire spurious DRDY interrupts.
  - If SYNC or CS is ever left Hi-Z (on the 10 k R78/R77 only), CLK or MOSI crosstalk rides on a high-impedance wire.
- *Fix (cheap, no layout change): wiring and firmware rules for §9/§10.*
  - Cable ≤ 10–15 cm.
  - MCU-side series R of 47–68 Ω on CLK, SCLK, MOSI and CS. Or keep 33 Ω and use the lowest GPIO speed that meets timing. Medium speed is enough for 5.12 MHz CLK and ≤ 8 MHz SCLK.
  - Drive SYNC and CS push-pull at all times; never use hardware-NSS Hi-Z.
  - Arm the DRDY interrupt only while CS is high.
- *Next revision:* add 0402 series-R footprints (0 Ω / 33 Ω) at J7 on SCLK/MOSI/CS/CLK, and consider a 2×7 or 2×8 header so that DRDY and SCLK each get a GND neighbour.

**S2. Nine DRC errors are not reported in DESIGN §5.11: `solder_mask_bridge` on J3.**
- *Location:* J3 pads 1–10, x = 125.389, y 101.05–105.55.
- *Evidence:*
  - Pads are 0.3 mm wide at 0.5 mm pitch with a 0.102 mm local mask expansion, so each aperture merges with its neighbour and there is no mask dam.
  - `PCB v2.kicad_pro` contains 9 exclusions for exactly these, but at x = 124.989 (stale, 0.4 mm off). They no longer match, and kicad-cli reports them as **errors**.
- *Impact:* SI impact is benign. Every J3 neighbour pair (GND / S1…S8 / GND) is at about 0 V (virtual grounds), so leakage across flux residue carries no current. The practical risk is bridging while hand-soldering, and the fab/CAM flagging it.
- *Fix:* re-exclude the items in pcbnew (or set the J3 pad mask margin to 0), and note it in §5.11. Tell the fab that no dam is expected on J3. Wash the flux (already in §9.2).

### NOTE

**N1. Sensor (high-Z) inputs are clean; the DESIGN §2.8 claims are confirmed with measured numbers.**

*Layer and via evidence.*
- S1–S7, S8 and S_sac: 100 % F.Cu, 0 vias, no B.Cu copper.
- 100 % of their length lies over solid In1 GND. In1 has no void larger than 1 mm² anywhere on the board.
- Lengths (mm): S1 27.2, S2 24.9, S3 11.0, S4 13.2, S5 12.8, S6 10.6, S7 23.5, S_sac 17.1, S8 6.9.
- Stray capacitance to GND: ≤ 27 × 0.079 ≈ 2.2 pF per channel, which is negligible against the PVDF capacitance.
- Only own-channel Q copper comes within 0.3 mm of an S node (the Rf/Cf pads). That stray adds to Cf, a gain change well under 0.1 %.

*Closest cross-channel approaches to an S node (edge gap; how much of the S trace lies within 1 mm; share of the gap covered by the F.Cu guard):*

| Victim ← aggressor | Location | Gap | S trace within 1 mm | Guard share |
|---|---|---|---|---|
| S5 ← Q6 | C15 pad 2 / Q6 trace at x 129.78, S5 at y 103.35 | 0.77 mm | 2.2 mm | 0 |
| S7 ← Q6 | | 1.13 | | 0.70 |
| S4 ← Q3 | | 1.18 | | 0.29 |
| S_sac ← Q7 | | 1.20 | | 0.67 |
| S1 ← Q2 | | 1.22 | | 0.67 |
| S4 ← Q6 | | 1.27 | | 0 |

- Worst case, S5 ← Q6: Cc ≈ 2.2 mm × 0.25 fF/mm plus pad fringe, **≤ 2 fF**. That gives Cc/Cf ≤ −78 dB above 10.6 kHz, and ωCcRf ≈ −98 dB at 1 kHz.
- No Vout, D-node, digital or charge-pump net is within 2 mm of any S net.
- Nearest digital to an S net: SCLK → S1 at 26.7 mm. Nearest charge-pump node (C+/C−/CP) to an S net: 13.2 mm.
- The worst on-board crosstalk mechanism is S-to-S: S1∥S2 for 20 mm at a 0.2 mm gap, about 66 fF. It only matters when a channel saturates and its −IN leaves virtual ground: 66 fF / 15 pF gives about 0.4 % of the swing during the clip. The FFC itself couples far more.

**N2. Charge pump: acceptable.**
- The switching nodes (Net-(U4-C+), Net-(U4-C−), Net-(U4-CP)) fit in a bbox of 147.0–154.0 × 84.1–87.6.
- Loop and cap distances:
  - flying cap C6 is 1.5 mm from pins 9/10;
  - C33 (VIN) is 1.9 mm from pin 3; C34 is 3.2 mm;
  - C17 (CP) is 2.8 mm from pin 5;
  - C7/C9 are 1.5/1.9 mm from pins 6/11;
  - every cap GND pad has its own via within about 1.0–1.3 mm;
  - the U4 EP uses the footprint's thermal vias.
- Nearest analog copper to the switching nodes:
  - +3.3VA route: 3.2 mm (F.Cu);
  - Q4: 5.3 mm (B.Cu, shielded by In1 and In2);
  - Vout1: 5.6 mm;
  - AIN0P: 5.8 mm.
- Everything is over unbroken In1, so coupling is under 0.01 fF/mm. The 21 starved thermals include U4 pins 3, 6, 9, 10 and 11 at one spoke each. The spoke is only 0.2 mm long, so it has no electrical effect (known item §9.11).

**N3. AIN fan-in and U2 grounding: good.**
- AIN0–7P: F.Cu only, 0.1 mm, 0 vias, 100 % over In1. Lengths are 16.2 / 9.8 / 9.8 / 6.2 / 9.3 / 6.5 / 12.7 / 12.8 mm; the length mismatch does not matter at 10 kSPS.
- AINxN go straight into the GND bar inside the pad ring (21 GND stubs, 0.2 mm).
- **Three** GND vias under U2, at (165.52, 102.11), (166.1, 101.0) and (166.1, 103.2). DESIGN §5.7 says "two".
- The pseudo-differential reference is therefore In1 at U2, while each C_AA/R_bot returns to In1 through its own via-in-pad about 5–10 mm away. With a void-free plane this is fine.

**N4. Decoupling distances (IC pin → cap pad; cap GND pad → GND via).**

| IC pin | Cap | Pin → cap | Cap GND → via | Notes |
|---|---|---|---|---|
| U2 AVDD pin 15 | C37 | **2.84 mm** (0.15 mm track) | via-in-pad | The return to AGND pin 13 runs about 6 mm through In1 to the vias under U2. With ~3–4 nH, 1 µF resonates near 2.5 MHz ≈ f_MOD (estimate). TI §11.1 asks for "as close as possible". An 0402 100 nF directly at pins 15–13 would be better next revision. |
| U2 DVDD pin 26 | C36 | 1.85 mm | 0.08 mm | |
| U2 CAP pin 24 | C35 | 2.05 mm | via-in-pad | |
| U2 REFIN pin 14 | C59 | 1.93 mm | via-in-pad | |
| U3 VCC | C58 | 1.63 mm | via-in-pad | |
| U7 IN | C60 | 1.73 mm | 0.94 mm | |
| U7 OUT | C61 | 1.65 mm | 1.05 mm | |
| Op-amp V+ (pin 4) / V− (pin 13), U6/U11/U12/U13 | C22–C28 / C11 (B.Cu) | 1.58–1.62 mm of F.Cu to a 0.6/0.3 via-in-pad | 0.5–1.26 mm | |

- DESIGN §3.6's wording "the +2V5 pads reach the In2 plane through about 0.1 mm of prepreg" is misleading. The connection is the via-in-pad at C22/C23/C25/C27; it was verified in the fill.

**N5. FB1 split and supply paths: adequate.**
- *+3.3VA:* U7 → C61 → 0.4 mm F.Cu → via (152.48, 92.28) → 0.4 mm B.Cu along y 91 and x 164.8 → via (169.3, 106.7) → C37 → U2 pin 15 and FB1 pin 1 at (170.51, 109.0). About 53 mm in total.
- *+3V3:* FB1 → R77/R78 → B.Cu at x 174.5 → U3/C58 → B.Cu at y 96.1 → C36 → DVDD.
- The split sits at the ADC, which is correct. The digital current shares the 53 mm +3.3VA feed up to C37, which is fine.
- *Current capacity:*
  - ±2V5 each feed their plane or pour through 5 × 0.6/0.3 vias: +2V5 at (157.4–159.9, 82.07); −2V5 at (145–146, 89–90.6). That is ample for about 0.1 A per rail.
  - The +2V5 feed is a 0.6 mm track from C20; −2V5 is 0.5 mm.
  - The 0.1 mm tracks on these nets are only the FB sense lines to R1/R2 and the U7 IN/EN (few mA).
  - +5V → U4 runs through zones and FB2 (1 A part).
  - Nothing is under-sized.

**N6. The digital bus on the board is fine.**
- *Routing:* 0.1 mm lines, F.Cu, 100 % over In1, 0 vias (except SYNC).
- *Lengths:* SCLK 39.3, CS 38.2, DRDY 35.8, MISO 30.6, MOSI 27.1 mm.
- *Edge gap:* 0.2 mm in J7's row gap; 0.3 mm along the right edge (x 174.9–176.5).
- *Coupled lengths:* DRDY–SCLK 31 mm, CS–DRDY 30 mm, MISO–SCLK 27–29 mm, MISO–MOSI 21 mm.
- *Crosstalk at t_r ≈ 1 ns (fast H7 / LVC, unverified):* line delay is about 0.17 ns over 30 mm.
  - NEXT ≈ Kb · 2Td / t_r ≈ 0.041 × 0.34 ≈ 1.4 %, about 45 mV.
  - FEXT ≈ |Kf| · Td / t_r ≈ 0.04 × 0.17 ≈ 0.7 %.
  - Both are harmless.
- *Critical length:* t_r / (6 · t_d) is about 30 mm at 1 ns and 60 mm at 2 ns. The on-board runs are marginal only with the fastest GPIO setting, which is another reason for S1.
- *Terminations:* R81 (DOUT) and R82 (DRDY) are 2.3 and 3.5 mm from U2's pins, i.e. source-terminated. Good.

**N7. The clock path is good.**
- *Route:* J7 pin 2 (171.38, 89.705) → MCU_CLK 9.2 mm, 0.15 mm, daisy-chained through R79 pad 1 (no stub) → U3 pin 2. Then U3 pin 4 → 2.0 mm → R80 → 2.9 mm → U2 pin 23.
- *Post-buffer run:* 4.9 mm, far below critical length. The 33 Ω sits at the driver, as TI §11.1 asks.
- *Aggressors:*
  - MOSI runs 0.35 mm from MCU_CLK (1.75 mm within 0.5 mm) and 0.47 mm from CLKIN (1.1 mm within 0.5 mm);
  - CAP runs 0.2 mm from CLKIN for 1.2 mm.
  - All victims here are low-impedance or buffered, so there is no glitch risk.
- DESIGN §4.1 says "3 mm run"; the actual post-buffer run is 4.9 mm (cosmetic).

**N8. The digital-to-analog separation claim in DESIGN §5.6 is wrong, but harmless.**
- DESIGN says "The lines stay ≥ 5 mm from any AIN trace". Measured:
  - ADC_SYNC_RESET → AIN7P: 1.70 mm;
  - ADC_MISO → AIN0P: 2.48 mm;
  - CLKIN → AIN0P: 2.73 mm;
  - SCLK → Vout4: 4.78 mm.
- *Coupling at 2.5 mm over a 0.1 mm plane:* ≪ 0.05 fF/mm, i.e. < 0.1 fF in total. That gives at most 0.33 fC per 3.3 V edge into a 10 nF AIN node, about 30 nV (below 1 LSB = 143 nV) and then filtered. Only the documentation needs correcting.

**N9. Return paths are good.**
- Every F.Cu signal segment lies over In1. The only exceptions are ≤ 0.5 mm where a track enters its own via antipad (≤ 1.0 mm for Vout1/Vout5/Vout6/Vout7, which cross one antipad).
- In1 has no hole > 1 mm².
- The B.Cu signals (Q4 20.9 mm, Vsac 68.8, Vout2/6/7 11–18, D-nets 1.5 each, SYNC 35.0 mm, +3.3VA, +3V3) reference In2 (+2V5). There is no GND stitching at their layer changes; the return must cross In1 → In2 through the nearest ±2V5 decoupling or the roughly 77 pF In1–In2 inter-plane capacitance (1.24 mm core, estimate).
- That is irrelevant for the kHz, low-impedance analog nets.
- The only digital net on B.Cu is SYNC_RESET: 2 vias, 0.51 mm from the right board edge at x 176.7, within the −2V5 pour.
  - It toggles only at start and resync, so it is acceptable.
  - Firmware should hold it push-pull high (see S1).

**N10. Documentation drift in DESIGN.md (fix the text only).**
- §5.6: "≥ 5 mm from AIN" is wrong (see N8).
- §5.7: "two vias" under U2; there are three.
- §4.1: "3 mm run"; the actual run is 4.9 mm.
- §5.2: "≈ 59 Ω, 0.10 pF/mm"; this review's solver gives 63–68 Ω and 0.08 pF/mm.
- §5.11: omits the 9 J3 `solder_mask_bridge` errors and the stale exclusions (S2).

These do not affect the ordering decision.

**N11. Stale outputs confirmed.**
- `PCB v2-*.gbr`, `PCB v2-job.gbrjob` (2026-09-16), `PCB v2.csv`, `distinta PCB v3-TESI.xlsx` and `bom/ibom.html` all predate the ADC swap.
- Regenerate them from the current board (known, §9.3).

---

## 3. Summary

- **Parity is exact:**
  - 132/132 footprints, 386/386 pads, 88 nets, 0 value/footprint/pad-net mismatches;
  - DRC schematic-parity 0, unconnected 0;
  - the BOM matches the PCB.
- **Consistent across schematic, PCB, DESIGN.md and silk:** the AIN channel map, the J7 pinout (pads and silk), J5 (pads and silk), the TP nets, and the AIN0–7 / CH1–7 silk.
- **No BLOCKER.**
- **SHOULD-FIX S1 (partly known):** the J7 ribbon/MCU link carries all the SI risk:
  - DRDY is flanked by MISO and SCLK;
  - SCLK/MOSI/CS have no damping at the receiver;
  - 33 Ω at the MCU under-terminates a ribbon.
  - Fix: a short cable, 47–68 Ω or slow GPIO at the MCU, SYNC/CS always push-pull, and DRDY interrupt gating.
- **SHOULD-FIX S2:** 9 J3 `solder_mask_bridge` DRC errors. The exclusions are stale (0.4 mm off) and §5.11 does not report them. Re-exclude, or set the J3 mask margin to 0, and tell the fab.
- **Analog SI is sound:**
  - sensor nets: F.Cu only, 0 vias, fully over In1;
  - worst cross-channel coupling (Q6 → S5, 0.77 mm): ≤ 2 fF, about −78 dB;
  - digital ≥ 26.7 mm from sensor nets;
  - charge-pump nodes ≥ 5.3 mm from any analog net, all over void-free In1.
- **Minor, next revision:** C37 (AVDD) sits 2.84 mm from pin 15 with a roughly 6 mm plane return (N4). Several DESIGN.md numbers are off (N8, N10).
