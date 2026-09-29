# Final pre-order review: PVDF e-skin acquisition board (PCB v2)

**Reviewed state:** 2026-09-29, branch `riccardo_fix`, commit `c884b04`.

**Reviewers.** Five independent reviews. Each one ran KiCad 10 checks on its own copy of the project. The detailed reports, with the evidence for every item below, are in [`review/`](review/):
1. [Requirements and scope](review/1_requirements.md)
2. [Schematic correctness](review/2_schematic.md)
3. [PCB manufacturability](review/3_pcb.md)
4. [Schematic ↔ PCB parity and signal integrity](review/4_parity_si.md)
5. [Silkscreen, hand assembly and testability](review/5_silk_dfa.md)

I re-checked the claims marked ✔ against the board file myself.

---

## Status after the designer's decisions (2026-09-29)

| Item | Status |
|---|---|
| B1 stackup | **Decision: order JLC's default stackup (7628, 0.21 mm outer prepreg).** The designer judged the thicker dielectric acceptable. The design's 0.1 mm-based crosstalk, guard and return-path figures (DESIGN §5.2) are roughly 2× optimistic, but still small. The board file still records the 0.1 mm stackup. |
| B2 stale outputs | The designer regenerates the fab outputs from the current board. |
| S1 op-amp pin 1 | **Fixed by the designer.** The Opa4 silk pin-1 marker now sits at pin 1; the stray bar and both `silk_overlap` items are gone. |
| S2 mask dams | **Fixed.** Removed the local 0.102 mm pad mask margin on J3 and U6/U11/U12/U13 (76 pads), which now inherit the board's 0. Dams are 0.20 mm (J3) and 0.25 mm (op-amps). The 9 `solder_mask_bridge` DRC errors are gone. |
| S3 wrong-part labels | **Partly done.** The designer re-placed the C9, C19, R1 and R6 labels. The R6 label is still closer to R1 than to R6, and the C34 label still sits over FB2. Check these two against the iBOM when fitting them. |
| S4 back diff-amp labels | Unchanged. Assemble that block from the iBOM, one value at a time. |
| S5 knockout labels | **Fixed.** The 7 test-point and J5 labels are now normal (non-inverted) text, still 0.5 mm bold. |
| S7 small vias | **Fixed.** All vias now have a hole of at least 0.3 mm: 205 × 0.6/0.3, 8 × 0.45/0.3, and 1 × 0.4/0.3 (U4 pin 8 EN+, boxed in between the WSON pins and C6). They fit JLC's cheapest class, "0.3mm/(0.4/0.45mm)". The smallest hole on the board is 0.3 mm. |
| S6 identification | **Done by the designer.** B.Silk carries "Transimpedance Amplifier PVDF Interface / Rev 1 / Designers: Gabriele Odino - Riccardo Testa". The UniGe/DITEN logo is on F.Silk (bottom edge) and the COSMIC lab logo on B.Silk. Both logos are board-only and excluded from the BOM and position files. The back AIN0–7 labels were removed. |
| Logo line widths | The fine lines won't print cleanly. Checked by eroding the silkscreen render: DITEN's three-line department name has strokes of about 0.10–0.12 mm, and the COSMIC tagline ("Connected Objects / Sensing Materials / Integrated Circuits") about 0.07–0.08 mm. JLC's minimum silk line is 0.15 mm, so expect the department name to print faint or broken and the tagline to be lost. The university name, the shield, "DITEN", "COSMIC lab" and the title text are wide enough. Cosmetic only. |
| S8–S11 | Open (optional). |

**DRC after these fixes:** 0 unconnected, 0 clearance errors, 0 mask bridges, 0 silk overlaps. The only items left are the intentional ones: 63 `text_height` (0.5 mm text), 21 starved thermals, 22 + 5 library notices, and the non-physical C48/J7 courtyard overlap.

In the order settings (§4), use **no specified stackup** (JLC default) and **min via 0.3mm/(0.4/0.45mm)**.

---

## Verdict

**The design is electrically correct and ready to order, once you take care of two order-process blockers.** No reviewer found a design-file blocker.

| Area | Result |
|---|---|
| Circuit versus intent | The netlist implements the documented chain exactly: 7 channels plus the reference, diff-amp polarity, ±2.50 V rails, 10 kSPS, and ADC inputs within limits even with op-amps saturated. |
| Schematic | ERC shows 0 errors; all 31 warnings are harmless (library tables, symbol copies, one label notice). Every IC was checked pin by pin against its TI datasheet. |
| Schematic ↔ PCB | 132/132 footprints, 386/386 pads and 88 nets match, with 0 mismatches. DRC parity finds 0 issues and 0 unconnected items. The AIN map, the J7/J5/test-point labels and the AIN/CH labels all match the nets. |
| PCB fab | 0 clearance, short, hole or edge errors. Everything is within the JLCPCB 4-layer standard. The In1 GND plane is solid under every SPI, CLK, AIN and sensor trace. The ±2V5 pours have no islands. |
| Signal integrity | Sensor inputs are on F.Cu only, with no vias, over solid GND. The worst cross-channel coupling is about −78 dB. Digital lines are ≥ 26.7 mm from the sensor nets. On-board SPI crosstalk is about 1 %. |

---

## 1. BLOCKERS: the order itself (no board change)

| # | Item | What to do |
|---|---|---|
| **B1** | **Stackup.** The design assumes a 0.1 mm outer dielectric. JLC's default stackup (7628) is **0.21 mm**. | Tick *Specify stackup* and choose **JLC04161H-3313** (0.0994 mm prepreg, 1.265 mm core). |
| **B2** | **The Gerbers, job file, `PCB v2.csv`, `.xlsx` and `bom/ibom.html` in `PCB/` are stale.** The Gerbers and job file describe an old **2-layer 53.75 × 40.72 mm** board, with no inner layers and no drill file. | Do **not** upload them. Delete them and regenerate from the current board (§5). A test export on a copy came out correct: 4 layers, 56.1 × 45.1 mm. |

---

## 2. SHOULD-FIX before ordering: small board or schematic edits

All of these are quick. I can apply any of them with the usual snapshot → offline DRC → apply workflow.

| # | Item | Why it matters | Fix | Effort |
|---|---|---|---|---|
| **S1** | **The op-amp pin-1 mark is misleading** on U6, U11, U12 and U13 (Opa4 footprint). ✔ | The pin-1 notch sits on the body edge line, so the IC hides it. The only mark visible outside the body is a bar next to the **pin 9/10 corner**, e.g. U11 at (152.7–153.7, 99.26), which looks like pin 1. Fitting the part 180° round puts the supplies on the inputs and kills a quad (4 channels). The same bar causes both DRC `silk_overlap` items. | Delete the stray bar and add a pin-1 dot outside the body next to pad 1, in all four footprints. | 5 min |
| **S2** | **Solder-mask dams missing** on J3 and on the four op-amps. | The pads carry a local mask margin of 0.102 mm. That leaves −0.004 mm between J3 pads (the 9 DRC `solder_mask_bridge` errors) and 0.05 mm on the op-amps; JLC's minimum is 0.1. With no dam, a bridge between sensor-input pins is easier to make and hard to see. | Set the pad mask margin to 0 in the 5 footprints. That gives dams of 0.20 mm (J3) and 0.25 mm (op-amps). | 5 min |
| **S3** | **Four references sit on the wrong part** (my 0.8 mm placement pass). ✔ | R6's label is closer to R1 (0.67 vs 2.12 mm). These are 130k and 120k in the charge-pump feedback, so a swap shifts +2V5. C34's label sits over FB2, C19's over R9, and C9's is equidistant from R1/C20/C9 (100n vs 4.7µ). | Re-place those four labels. | 10 min |
| **S4** | **Back diff-amp labels are ambiguous.** ✔ | About 12 of the 0.5 mm labels (e.g. R23/R24, R25/R26, R29/R28, R33/R41, R43/R44, R48/R51) are only 0.01 mm closer to their own part than to the neighbour. Neighbours alternate between 4.99k and 10k 0.1 % thin-film parts, which may be unmarked. A swap silently breaks that channel's gain and the reference rejection. | Centre each label over its own resistor column, **and** assemble this block from the iBOM, one value at a time. | 15 min |
| **S5** | **Knockout test-point and J5 labels are too small** (0.5 mm, 0.125 mm stroke). | JLC's minimum is 1.0 mm / 0.15 mm, PCBWay's 0.8 / 0.15. In knockout text the letters are the gaps, and they fill in. | Make these 7 labels plain (not knockout) text at 0.8 mm / 0.15 mm. | 10 min |
| **S6** | **No board identification.** | Revisions can't be traced, and JLC prints its order number at a random free spot, possibly in the dense blocks. | Add e.g. "PVDF e-skin ADC v2 · 2026-09 · <author / lab>" on B.Silk. Add a `JLCJLCJLCJLC` placeholder in a free area (e.g. front, x 160–172, y 113–120), or pay to remove the number. | 5 min |
| **S7** | **Six 0.25 / 0.15 mm GND vias** near J3 and J5. | These are the only 0.15 mm holes, and they trigger JLC's small-hole surcharge. Their annular ring is at the absolute minimum. | Change them to 0.3 / 0.2 mm. Tested on a copy: no new DRC items. | 5 min |
| **S8** | **Mounting holes: no copper keep-out.** | The B.Cu −2V5 pour starts 0.2 mm outside the GND pads of H1–H4, and S_sac passes 2.45 mm from H3's centre. A metal nut, washer or standoff can short −2V5 to GND through the mask. | Add ~3 mm-radius copper keep-outs on F.Cu and B.Cu, **or** use nylon hardware only (note it in the assembly notes). | 15 min, or none |
| **S9** | **The 0.1 % diff-amp resistors share the value "10k"** with the 1 % pull-ups R77/R78. | A regenerated KiCad BOM merges them into one line (reproduced). If 1 % parts get fitted, the worst-case CMRR drops from about 57 to about 37 dB. | Set Value to `10k 0.1%` (or add a Tolerance field) on the 14 precision 10k parts. Do the same for the 4.99k parts. Schematic only. | 5 min |
| S10 | 21 starved thermals in the small charge-pump and USB pours. | Electrically fine: one 1 mm spoke per pad. They remain as DRC errors. | Set those pours to *Solid* pad connection, or exclude the errors. | 5 min |
| S11 | C48/J7 courtyard overlap. | Not physical: the pads are 0.88 mm apart and J7's body ends at y 89.77. | Add a DRC exclusion. Solder C48 before J7. | 1 min |

Optional: U4 (LM27762, WSON with exposed pad) is the only part that needs hot air or a hotplate. Opening the **back** solder mask over its exposed pad would allow iron-only assembly.

---

## 3. Decisions to take now: not board changes, but they affect what you order

| # | Decision | Recommendation |
|---|---|---|
| **D1** | **What must the front end measure?** 1 MΩ ∥ 15 pF gives a **dF/dt (current) response**. Taps, slip and vibration are fine, but slow or static pressure is not measurable: op-amp offset ≈ 0.5 nA ≈ 20 N/s of apparent dF/dt. | Confirm with your supervisor **before ordering parts**. The hedge is BOM-only, on the same pads: also order a set of **10 MΩ + 1 nF C0G** (true charge amp, 16 Hz corner, ~25 mV/N) or **1 GΩ + 220 pF**, for one board. Change all 8 channels together. |
| **D2** | **SPI clock.** Every ADS131M08 frame is 10 words and cannot be shortened while converting (DS §8.5.1.7, §8.5.1.11). At 1 MHz a frame takes 160–320 µs, against a 100 µs period at 10 kSPS. | Use **SCLK = 5.12 MHz** (at least 2.56 MHz) from the same H7 PLL that generates the 5.12 MHz MCO for CLKIN. The FOGLIO1 note's "~8 MHz" is not synchronous with it; update the note. |
| **D3** | **Word length and gain.** At gain 1, a 1 N / 10 ms press is about 2.4 mV at the ADC (~0.2 % of full scale), and 16-bit words add noise comparable to the ADC's own. | Use 24-bit or 32-bit words. Set the PGA gain per channel after the first measurements; gain 4–32 is electrically safe. Keep AIN7 (Vsac) at gain 1. |
| **D4** | **Power order and the J7 cable.** An H7 powered before this board back-powers U2 through CS/SYNC/SCLK/MOSI, and nothing limits the current (U2 is rated 10 mA per pin). In a ribbon, DRDY has no GND neighbour. | On the **MCU side**: 1 kΩ in series on CS and SYNC, 220–330 Ω on SCLK/MOSI/CLK. Keep those pins Hi-Z until DRDY reads high. Keep the cable ≤ 10–15 cm. Mark pin 1 (SYNC, top-right) on the cable, because J7 is unshrouded. |
| **D5** | **FFC orientation.** J3 is dual-contact, so a flipped flex still connects but mirrors S1↔S8. That puts a taxel on the reference amplifier. | Document the flex pinout and the FFC type (same-side vs opposite-side contacts). Buzz it out before first use. Also add the mating FFC to the BOM. |
| D6 | **Input protection:** none. The OPA4350 is rated only ±1 kV HBM, on a surface people touch. (known) | Make sure the touched outer electrode is GND/common. Handle the flex with ESD care. **Buy spare OPA4350s** (/2K5; the /250 reel is Last-Time-Buy). Next revision: 1 kΩ in series per input. |

---

## 4. Order settings (JLCPCB)

```
Layers ................ 4        Thickness ........ 1.6 mm      Material .... FR-4
Specify stackup ....... YES -> JLC04161H-3313  (outer prepreg 0.0994 mm, core 1.265 mm)
Impedance control ..... not required (if the form ties stackup choice to "yes", choose yes with no targets)
Outer / inner copper .. 1 oz / 0.5 oz
Min via hole/diam ..... 0.3mm/(0.4/0.45mm)   (all vias >= 0.3 mm hole, >= 0.4 mm diameter)
Surface finish ........ ENIG   (flat pads: 0.5 mm TQFP-32, WSON-12, FFC; easier hand soldering)
Solder mask / silk .... Green / White
Via covering .......... Tented  (NOT plugged / filled); vias inside SMD pads stay open by design
Order number .......... Specify location (JLCJLCJLCJLC text) or remove
Remark ................ "Stackup must be JLC04161H-3313 (0.1 mm outer dielectric).
                         Via-in-pad intentional for hand assembly: leave open, no filling/capping."
Stencil ............... not needed (hand assembly)
```

For PCBWay or another fab, state: 4 layers, 1.6 mm; outer prepreg ≈ 0.1 mm (1080, 2116 or 3313 class) on both sides; core 1.2–1.3 mm; ENIG; tented vias with via-in-pad left open; minimum drill 0.2 mm.

---

## 5. Outputs to generate (after the fixes; from pcbnew, or kicad-cli on a copy)

1. **Board finish:** in *Board Setup → Board Stackup*, set the finish to ENIG, so the job file records it.
2. **Gerbers:**
   ```
   kicad-cli pcb export gerbers --check-zones -l "F.Cu,In1.Cu,In2.Cu,B.Cu,F.Mask,B.Mask,F.Silkscreen,B.Silkscreen,Edge.Cuts" --subtract-soldermask -o fab/ "PCB v2.kicad_pcb"
   ```
3. **Drill files:**
   ```
   kicad-cli pcb export drill --format excellon --excellon-units mm --excellon-separate-th --generate-map --map-format gerberx2 -o fab/ "PCB v2.kicad_pcb"
   ```
4. Zip `fab/`, including the `.gbrjob`, and check it in JLC's Gerber viewer. You should see 4 layers, 56.1 × 45.1 mm, and silkscreen on both sides.
5. **For assembly:**
   - a regenerated **InteractiveHtmlBom**, which is essential for the 0.5 mm dense blocks;
   - F and B assembly PDFs;
   - the Mouser BOM.

   No pick-and-place file is needed.

---

## 6. Hand-assembly plan (from review 5)

1. **U4** (WSON with exposed pad): hot air or hotplate on the bare board. Tin the exposed pad first; its 3 open vias wick solder.
2. **U2** (TQFP-32, no thermal pad): drag-solder with an iron.
3. **U6, U11, U12, U13:** **check pin 1 against pad 1 (top-left), not the silk bar** (S1). Then U3 and U7.
4. **J3** (0.5 mm pitch): inspect for bridges with a loupe.
5. **Front passives:** C48 before J7. There are 62 open vias in pads: pre-tin the pads and use a little extra solder.
6. **J2** SMD pins.
7. **Back side,** with the board in a holder:
   1. U1, then the passives.
   2. The **diff-amp block from the iBOM, one value at a time** (S4).
8. **J2** shell legs.
9. **J5 and J7** last.

Use a 60 W-class iron for the GND pads inside the AFE guard pour, which are solid-connected. Scope ground: the test points won't take a clip, so use the plated mounting holes (GND).

---

## 7. Bring-up and firmware notes

- **Power first, then the H7** (D4).
- The ADC needs CLKIN to run (no crystal). Without the H7, a 5.12 MHz 3.3 V square wave plus a USB-SPI bridge on J7 can exercise it.
- Probe points:

  | Signal | Probe at |
  |---|---|
  | Vsac | R75 pad 1 / C8 pad 2 |
  | Vout_n | the 1.1 k divider tops |
  | AIN | C48–C55 |
  | CLKIN | R80 |
  | DRDY | R82 |

- **Offset checks:** J5 on 1–2 grounds S_sac, giving Vout_n = −2·Q_n. The ADC's input mux can short each channel and apply test signals. Global chop cuts the ADC offset to about ±32 µV.
- **Never enable current-detect mode:** REFIN has a capacitor.
- Channel map: AIN0=Vout1, AIN1=Vout4, AIN2=Vout2, AIN3=Vout3, AIN4=Vout5, AIN5=Vout6, AIN6=Vout7, AIN7=Vsac.
- AIN7 records Vsac, so firmware can rebuild the raw channels and try weighted subtraction.
- If a fixed spur shows with shorted inputs, it is probably a charge-pump harmonic (1.7–2.3 MHz) near f_MOD = 2.56 MHz. Shift OSR or CLKIN to confirm.
- With J7 unplugged, SCLK/DIN float and draw some extra ADC current; this is harmless. For the next revision, add 100 k pull-downs.

---

## 8. Next revision (not for this order)

- 1 kΩ series resistors (plus clamps) on the sensor inputs.
- Series-resistor footprints and more GND pins at J7, or a shrouded header.
- 100 k pull-downs on SCLK/DIN.
- A 0402 100 nF directly at U2 pins 15–13: C37 sits 2.84 mm away, with a ~6 mm return.
- Test points on Vsac, +3.3VA and CLKIN.
- A DNP crystal-oscillator footprint.
- In2 clearance 0.25–0.3 instead of 0.5.
- An LM27762 PGOOD output to J7.
- A board-level 5 V surge margin: D1's standoff is 5.0 V and the LP5907 VIN max is 5.5 V.

---

## 9. DESIGN.md corrections found by the reviewers (documentation only)

- ADC input impedance is **528 kΩ** at this clock, not 330 kΩ. XTAL_DIS = 1 is recommended for power, not required. U2 pin 27 is NC (grounding it is allowed).
- The closest digital-to-AIN spacing is **1.70 mm** (SYNC to AIN7P), not "≥ 5 mm"; the coupling is still negligible.
- There are 3 GND vias under U2, not 2.
- The clock run after R80 is 4.9 mm, not 3 mm.
- Microstrip: about 63–68 Ω and 0.08 pF/mm, not 59 Ω / 0.10 pF/mm.
- The FOGLIO1 J7 note should say SCLK 5.12 MHz, not "~8 MHz" (D2).
- The J3 `solder_mask_bridge` exclusions in the project file no longer match the markers (0.4 mm off). S2 removes the need for them.
