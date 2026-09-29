# Review 5: silkscreen, hand assembly (DFA), testability, ordering outputs

**Board:** `PCB v2.kicad_pcb` on `riccardo_fix` @ c884b04, reviewed on 2026-09-29.
**Method:** I ran everything on a copy in `review/5_silk_dfa/`:
- `kicad-cli` 10.0.6 DRC (JSON);
- per-layer B&W SVG → 1000 dpi PNG, then pixel ANDs of silk against mask openings;
- a fresh Gerber export in `5_silk_dfa/g/`;
- a pcbnew-python dump of footprints, pads, texts, zones and vias;
- colour composites in `5_silk_dfa/r/`:
  - `comp_F.png` and `comp_B.png` (B not mirrored);
  - `compfab_*.png`, which add Fab and courtyard;
  - the crops cited below. B crops are mirrored, i.e. seen from the back.

Coordinates are board mm.

**Tally:** 1 conditional BLOCKER (stale outputs), 7 SHOULD-FIX (S1–S6, plus producing the ibom and assembly PDF), 15 NOTE.

---

## 1. Silkscreen correctness

### What checks out
- **Silk over copper.**
  - DRC reports **0 `silk_over_copper`**. The 2 `silk_overlap` items are S1 below.
  - The pixel AND of silk against mask openings, edge excluded, gives **0 px on both sides**.
  - Nothing is within 0.05 mm of an opening. On B nothing is within 0.10 mm.
- **B-side text is mirrored correctly** (`fullB.png`, mirrored view: all text reads normally).
  - Every B text has `mirror=True`.
  - Angles are only 0° or 90° on each side, so there are **two reading directions per side**.
  - Vertical text on F reads bottom→top. On B it reads the same way physically (towards the J7/U2 edge).
- **J7 pin names match the pad nets exactly** (`j7.png`):
  - Pad 1 at (171.38, 84.655) is ADC_SYNC_RESET → "SYNC" at (171.38, 80.19).
  - Pads 3/7/11 are GND → "GND" at x 168.84 / 163.76 / 158.68, top row.
  - 5 CS → "CS" at 166.30. 9 DRDY → "DRDY" at 161.22.
  - 2 MCU_CLK → "CLK" at (171.38, 91.91). 4 MOSI → 168.84. 6 GND → 166.30. 8 MISO → 163.76. 10 SCLK → 161.22. 12 GND → "GND" at (158.68, 92.81).
  - The pin-9 net is ADC_DRDY, which reaches U2 pin 18 through R82 (U2_DRDY).
- **J5 labels match** (`j5.png`):
  - Pad 1 (124.58, 110.48) is GND → "GND" at (123.38, 108.48), above it.
  - Pad 3 (127.12, 110.48) is S8 → "S8" at (128.12, 108.48), above it.
  - Pad 2 (125.85, 113.48) is S_sac → "S_SAC" at (124.10, 113.73), left of it.
- **TP labels match their nets** (`tp.png`): TP1 = +5V, TP2 = +2V5, TP3 = −2V5, TP5 = GND.
- **AIN0–AIN7 (B) sit behind the right triples.**
  - For every k, the AINk label x is within 0.25 mm of the column of C(48+k)/R(61+2k)/R(62+2k), and its y is on the middle part R(61+2k). For example, AIN0 is at (156.13, 93.81) and R61 at (156.08, 93.81); AIN7 is at (159.33, 110.26) and R75 at (159.28, 110.26).
  - The nets confirm it: R(61+2k)–C(48+k)–R(62+2k) all carry AINkP. The Vout sources are 1, 4, 2, 3, 5, 6, 7 and Vsac, matching DESIGN §5.10.
- **CH1–CH7 (B) are next to the right groups** (`b1.png`, `b2.png`).
  - Nets confirm each group is D_nP/D_nN/Vout_n/Q_n for n = CH number.
  - One weak spot: CH3 at (150.43, 103.66) is 2.5 mm below R39/R40 and sits beside the R49 reference, which belongs to CH5. This is readable but crowded (NOTE N6).
- **BOM ↔ board coverage.**
  - All 124 BOM refs exist on the board, and every value matches.
  - The board has only H1–H4 and TP1/2/3/5 beyond the BOM (correct), plus the BOM's "(J5 jumper)" line.
  - The quantities per line match the ref counts.
- **Polarity:**
  - **D1** (SMF5.0A, unidirectional): pad 1 at (132.86, 85.47) is +5V, the cathode. The silk box is closed on the pad-1 (right) side, and the Fab arrow points to it (`usb.png`). Correct.
  - **D2** (LED): pad 1 at (141.89, 78.30) is GND, the cathode. The silk box is closed on the left, pad-1 side (`cp2.png`). Correct.
  - There are **no electrolytic or tantalum parts** (the BOM is all MLCC).
- **U2, U3, U7, U4 pin-1 triangles are outside the body** and stay visible after placement (`u2.png`, `cp2.png`).
  - U2 is at (≈162.8, 99.2) above pad 1 (162.97, 100.35).
- **J7 pin 1** has the KiCad SMD-header marker (an outline jog beside pad 1) plus the "SYNC" label.
- **J5 pin 1 does not matter.** On a 1×3 SMD header with alternating tails, Pin1Left and Pin1Right are the same land pattern turned 180°: pins 1 and 3 swap and pin 2 stays in the middle. Header posts are unpolarised, and the labels follow the pads. DESIGN §6 is correct.

### S1. SHOULD-FIX: the Opa4 footprint's pin-1 notch is hidden under the IC, and a stray silk line marks the wrong corner (U6, U11, U12, U13)
- **Location:**
  - U11 line at (152.68–153.69, 99.26), next to **pad 10**. Pad 1 is at (146.25, 95.47), top-left.
  - U12 line at (152.56–153.73, 109.9).
  - U6 line at ≈(130.8, 99.1–99.9).
  - U13 line at ≈(134.4, 105.8–106.6).
- **Evidence:**
  - Footprint graphics (pcbnew): `F.Silkscreen Line (153.69,99.26)-(152.68,99.26)` and `Arc (149.22,95.22)-(148.61,95.22)`.
  - The notch arc dips *into* the body. The body edge is at y 95.22, and the arc reaches ≈95.54 (`u11.png`, `u6.png`).
  - The line is on the side opposite pin 1. It is the only mark left visible after placement.
  - The two DRC `silk_overlap` items (R37/U6 at (130.78, 99.91) and U13/R47 at (134.43, 105.79)) are this same line.
- **Impact:**
  - After soldering, no visible mark confirms pin 1.
  - The one visible mark looks like a classic "pin-1 bar" at the pin-9/10 corner, which invites a 180° placement.
  - On the OPA4350, a 180° turn maps pin n to pad n+8. V+ (pin 4) and V− (pin 13) land on the GND +IN pads, while +INB and +INC land on the −2V5 and +2V5 pads. The inputs are then driven through their ESD diodes with the part unpowered, which will likely damage it (OPA4350 is LTB).
- **Fix:**
  - In the embedded Opa4 footprint, delete the stray line.
  - Add a Ø0.3–0.4 mm silk dot outside the body next to pad 1: local ≈(−3.6, −2.24) before rotation.
  - Update all 4 instances. This also clears both DRC silk_overlap items.

### S2. SHOULD-FIX: some references sit on the wrong part (contradicts DESIGN §5.10, "closer to that part than to any other")
The distance is measured from the text bbox to the part's pad bbox. Crops: `cp2.png` and `fullF.png`.

| Ref (value) | Label at | Its part | Nearer or equal part | Risk |
|---|---|---|---|---|
| **R6** (120k) | (152.31, 79.67) | R6 (151.11, 82.27): 1.39 mm | **R1 (130k): 0.00 mm**, the label is on R1 | R1↔R6 swap. They are the LM27762 FB divider, so the rail would be off by about 8 % |
| **C9** (100n) | (154.37, 78.19) | 2.24 mm | **C20 (4.7u): 2.24 mm**, R1: 1.80 mm | C9↔C20 are unmarked MLCCs of the same size |
| **C34** (10u 0805) | (145.45, 82.23) | 1.54 mm | **FB2: 0.03 mm**, C62: 0.38 mm | The label is stacked above "FB2", over FB2 |
| **C19** (4.7u) | (153.49, 90.23) | 0.77 mm | **R9 (100k): 0.00 mm** | The label is under R9 |
| FB2 | (145.69, 81.07) | 1.19 mm | R7: 0.56 mm | minor |
| U4 | (154.99, 84.28) | 2.70 mm | C20: 0.67 mm, C6: 0.95 mm | The label is on the far side of the C9/C6 column; the package is unique, so the risk is cosmetic |
| D1, C33, C7, C17, R22, R47, R57 | — | about equal to a neighbour | a cap↔resistor or size-mismatch pair | low, because the parts look different |

- **Fix:** move the labels as follows.
  - R6 → right of R6, about (153.0, 82.27).
  - C9 → above C9 only, about (153.57, 80.6), or vertical to its left.
  - C34 → below or right of C34, about (146.05, 86.6).
  - C19 → left of or below C19, about (150.19, 90.3), clear of the C17 label.
  - U4 → above U4, about (151.1, 82.9).
  - Recheck each against the vias, since KiCad is open.

### S3. SHOULD-FIX: B-side diff-amp references sit in the gap between 4.99k and 10k neighbours
- **Location:** B block, `b1.png`, `b2.png`, `b3.png`.
  - Labels are offset 0.70–0.90 mm sideways from their part's x. The column pitch is 1.46 mm and the pad gap 0.51 mm.
  - Examples:
    - R25 part x 145.18 → label 144.48;
    - R26 143.72 → 142.87;
    - R23 146.64 → 145.79;
    - R24 148.10 → 147.40;
    - R35 152.66 → 153.21;
    - R36 154.12 → 155.02.
  - Each 0.5 mm-wide vertical label therefore lands centred over the inter-column gap, above the column ends.
  - Pairs with *different* values and equidistant labels (own/other gap ≈ 0.86/0.87 mm):
    - R23/R24, R25/R26, R28/R29, R30/R31;
    - R33/R34, R35/R36;
    - R43/R44, R45/R46;
    - R48/R51.
- **Impact:**
  - The 4.99k and 10k parts are identical 0.1 % thin-film 0603s. Whether RT0603 parts carry a marking is **UNVERIFIED**; assume they are unmarked.
  - A swap silently changes a channel's gain from 2 and ruins its CMRR, which is the whole point of the sacrificial-sensor subtraction.
  - The parts are also 0.5 mm text on the back, which makes this worse.
- **Fix:**
  - Set each B-block reference x equal to its part x (0.5 mm text fits over a 0.95 mm pad), keeping the current y.
  - Otherwise, assemble strictly from the ibom (Section 6), one reel at a time: place all 14 × 4.99k first, then all 14 × 10k.

### S4. SHOULD-FIX: the 0.5 mm knockout user texts are below both fabs' minimums
- **Location:** 7 `gr_text` items at 0.5 mm, bold, stroke 0.12, knockout:
  - +2V5 (122.70, 92.56);
  - +5V (122.78, 96.53);
  - −2V5 (126.36, 84.14);
  - GND (126.36, 87.89);
  - GND (123.38, 108.48);
  - S_SAC (124.10, 113.73);
  - S8 (128.12, 108.48).
- **Fab limits:**
  - **JLCPCB:** minimum character height **1.0 mm (40 mil)**, line width **≥ 0.15 mm**, width:height **1:6**, and the same 1:6 ratio for hollow (knockout) text. Pad-to-silk clearance **0.15 mm**. Source: [JLCPCB capabilities](https://jlcpcb.com/capabilities/pcb-capabilities).
  - **PCBWay:** "Characters of less than 0.8mm high will be too small to be recognizable", "less than 0.15mm wide will be too narrow", best ratio 1:5. Source: [PCBWay capabilities](https://www.pcbway.com/capabilities.html).
- **What happens at 0.5 mm:**
  - Positive 0.5/0.12 text prints but blurs, because ink spread is roughly 0.03–0.08 mm per edge. Similar glyphs merge: 6/8/5/3, R/B.
  - In **knockout** text the strokes are *gaps* of about 0.12 mm in an ink field. Ink spread closes them, so the result is often a white rectangle with faint letters.
  - The TP and J5 labels are the ones you need at bring-up.
- **Fix:** make these 7 texts normal (not knockout), 0.8 mm / 0.15 mm. There is room:
  - to the right of TP2/TP1, as already done for TP3/TP5;
  - above and left of J5.

### N1. NOTE: the 0.5 mm / 0.12 mm references (56 of them: the divider block, the B diff-amp block and C23–C26)
- They are below both fabs' minimums (S4). Expect them to be legible only under magnification.
- The front divider block is well ordered (`div.png`: each ref on its own row, left column labelled on the left, right column on the right), so blur is tolerable there.
- If any room exists, use 0.6 mm / 0.15 mm. The stroke matters more than the height.
- Rely on the ibom for assembly regardless.
- The 0.8/0.15 references meet PCBWay's figures and JLC's stroke figure, but not JLC's 1.0 mm height advice. They print fine in practice.

### N2. NOTE: some silk sits within 0.15 mm of a mask opening
- JLC keeps silk 0.15 mm from pads, and may clip or shave it.
- The pixel scan found silk within 0.05–0.10 mm of openings at:
  - the **U11 ref** (x 144.86) and **U12 ref** (x 144.82), 0.09 mm from the pad-1 row opening at x 145.35;
  - the **C59 ref** (166.5, 109.3);
  - the **R74 ref** (155.0, 109.9);
  - the **R65 ref**;
  - the **"CLK"** label;
  - the **C49/C51/C53/C55** refs at x 160.63.
- Within 0.15 mm: the TP1/2/3/5 footprint circles (library-standard), and on B the R33, R36, R49, C23, C25 and C27 refs.
- **Fix:** nudge the U11/U12 refs 0.1 mm to −x. The rest are harmless if clipped.

### N3. NOTE: labels hidden after assembly
- The **J7** reference at (165.03, 87.18) is at the centre of the header, under the plastic body. The pin names identify J7 anyway.
- **J3**'s pin-1 circle at ≈(124.4, 101.05) is under the housing (`j3j5.png`).
  - The connector cannot be placed wrongly: the Z1/Z2 tabs at x 122.89 are opposite the contacts at x 125.39.
  - Reversing the flex only swaps S1↔S8 in mapping, since pins 1 and 10 are both GND.
  - Optional: a "1" next to pad 1, outside the housing, at about (126.3, 100.5).
- **U1** (B, SOT-323): its pin-1 triangle is under the body (`u1.png`). The 2-pad/1-pad geometry fixes the orientation, and pins 1/2 are interchangeable ESD channels. OK.
- **J5 "S_SAC"**: its bbox (122.77–125.43, 113.16–114.30) is 76 % inside J5's courtyard, beside pad 2's tail.
  - Whether the M50-363 housing covers it is **UNVERIFIED**. The housing width is not in hand, but a 1.27 mm single-row body should be about 1 mm wide around y 112, so the label is probably clear. Check the Harwin drawing.

### N4. NOTE: the U4 ref is far from U4, and the C7 label sits between C17 and C7
See S2. Cosmetic.

### N5. NOTE: the TP2/TP1 labels sit between the pads
- "+5V" at (122.78, 96.53) is 1.2 mm from TP2's pad and 0.25 mm from TP1's.
- It reads as TP1's label, but "above the pad" (TP2/TP1) and "right of the pad" (TP3/TP5) are mixed.
- **Fix:** fold into S4 by putting all four to the right of their pads.

### N6. NOTE: "CH3" at (150.43, 103.66) sits next to the R49 ref (CH5 group)
Consider (151.0, 98.3), between the R35/R36 and R39/R40 rows, if it is clear.

---

## 2. Polarity and orientation (summary)

| Part | Mark | Visible after placement | Verdict |
|---|---|---|---|
| U2 TQFP-32 | triangle outside the corner | yes | OK |
| U3, U7 SOT-23-5 | triangle | yes | OK |
| U4 WSON-12 | triangle at the top-left | yes | OK |
| U6/U11/U12/U13 SSOP-16 | notch inside the body plus a wrong-corner bar | **no / misleading** | **S1** |
| U1 SC-70 (B) | triangle under the body | no | OK by geometry |
| D1 TVS | box closed at pad 1 = K = +5V | yes | OK |
| D2 LED | box closed at pad 1 = K = GND | yes | OK. Confirm the part's cathode mark with a DMM diode test before soldering |
| J7 2×6 SMD | outline jog at pad 1, plus "SYNC" | yes | OK |
| J5 1×3 1.27 mm | none needed (symmetric) | — | OK |
| J2 USB-C | symmetric | — | OK |
| J3 FFC | dot under the housing | no | OK by geometry (N3) |
| Electrolytic / tantalum | none on the board | — | — |

---

## 3. Board identification

### S5. SHOULD-FIX: no board name, revision, date, author or university anywhere
- **Evidence:**
  - `texts.txt` lists only the J7, TP, J5, AIN and CH texts and the Cmts.User dimensions.
  - The board has **no `title_block`**.
  - Copper carries no text.
  - Two v2 builds (pre- and post-ADC-swap) will be indistinguishable.
- **Fix:**
  - **F.SilkS**, 1.0 mm / 0.15 mm, in the empty bottom-right area x 160–172, y 113–120 (`br.png`: no copper or silk there), for example at (166, 116.5):
    ```
    PVDF e-skin AFE  PCB v2 rev B
    ADS131M08 · 2026-10 · R. Testa · <University>
    ```
  - **B.SilkS**, 1.0 mm, in the large empty back area (x 124–140, y 95–118):
    - "BOTTOM – solder 1st";
    - the B parts list: diff-amp R 0.1 %, C11/22/27/28, R3/4/11/12, U1.
  - Optional **F.Cu** text "v2B" in the same bottom-right F.Cu void. That area has no pour, so the text is an isolated island; fine.
  - Fill the title block (Rev, Date, Company) too, so that plots and the ibom carry it.
  - **JLC order number:** JLC prints one unless you place `JLCJLCJLCJLC` (about 1 mm, e.g. on the B area) or pay for "remove order number". Otherwise it lands wherever JLC chooses, possibly over the B block.
  - An orientation hint is implicit (USB top-left, J7 top-right). The F/B labels above make it explicit.

---

## 4. Hand assembly (DFA)

- **Smallest parts:** all passives are **0603** (37 C, 67 R, 2 FB, LED), plus three 0805s. Nothing smaller. Fine-pitch parts:
  - **U4 WSON-12**: 0.5 mm pitch, 0.5 × 0.25 mm pads, with an exposed pad (EP);
  - **U2 TQFP-32**: 0.5 mm pitch, **gull-wing, no exposed pad** (32 pads, no pad 33);
  - **U6/U11/U12/U13 SSOP-16**: 0.635 mm;
  - **J3**: 0.5 mm pitch;
  - **U1 SC-70**.
- **U2 is iron-solderable.** TQFP-32 has gull-wing leads and no thermal pad. Tack two opposite corners, then drag-solder with flux and a 1.2–1.6 mm hoof or bevel tip, and wick the bridges.

### S6. SHOULD-FIX (optional): U4's exposed pad can only be soldered with hot air, because B.Mask is closed over its thermal vias
- **Evidence:**
  - U4 pad 13 has an F.Cu+F.Mask pad of 1.0 × 2.65, a **B.Cu-only** pad of 1.0 × 2.65 (no B.Mask), and three 0.6/0.3 mm `*.Cu` via-pads.
  - The fresh `g/PCB v2-B_Mask.gbr` has **no flash** near (151.09, 84.98). F_Mask has `X151090000Y-84980000D03`.
  - So the vias are tented on the back, and the "solder the EP from the back through the thermal vias with an iron" trick is impossible.
- **Impact:**
  - One part forces hot air (or a hotplate).
  - The LM27762 EP should be soldered: it is thermal and GND.
- **Fix:**
  - Add `B.Mask` to the embedded footprint's B.Cu EP pad.
  - The B side under U4 is free; R11/R12 are 3–4.6 mm away.
  - Then the whole board is iron-only: solder the pins with the iron, and feed solder into the vias from the back with a hot tip.
  - Otherwise keep hot air for U4 only.

### Other hand-assembly notes
- **N7. NOTE: iron access in the dense blocks.**
  - Minimum pad-to-pad gaps between different parts are **0.50–0.51 mm** everywhere:
    - divider rows (R61/R62, R63/R64, …);
    - B diff-amp columns (R28/R30, R55/R60, R43/R51, R35/R40);
    - charge pump (C7/R2, C19/R9, C61/U7, C62/FB2).
  - These are all the standard 0603-at-1.46 mm-pitch gap. Use a 0.8–1.2 mm bevel or chisel tip or a 0.4 mm conical tip, 0.3–0.5 mm solder, and tweezers. Good.
- **N8. NOTE: via-in-pad and plane coupling.**
  - There are **62 vias in SMD pads**, all 0.6/0.3 mm:
    - GND 26, +2V5 4, −2V5 4, the rest D/Q/Vout nets;
    - 19 F-side pads: C35–C37, C48–C55, C58, C59, R64–R76 and R79, U3, and U7 (×2);
    - the rest on the B block and C11/C22/C23–C28.
  - Vias are solid to their plane, so a GND/±2V5 via-in-pad is thermally a direct plane tie whatever the pad's relief. The plane reliefs themselves are thermal (In1/In2/B.Cu: 0.5 mm gap, 0.5 mm spoke).
  - The F.Cu `GND_AFE_GUARD` pour (124.6–136.6 × 86.6–120.4) is **solid** (pad connection = full). Solid pads: U6.3/.5/.12/.14, U13.3/.5/.12/.14 and **J3.1/.10**.
  - **Technique:**
    - Tin each via-in-pad pad first. A 0.3 mm barrel swallows about 0.1 mm³; keep feeding until the pad stays domed.
    - Then place the part and reflow onto the tinned pad.
    - Use about 360–380 °C on plane-tied pads.
    - A preheater (about 100 °C) makes the GND pads, J3.1/.10 and U4 much easier.
  - Vias are tented on the far side, so solder stops at the mask. Wicked solder does not reach the other side.
- **N9. NOTE: J3 FFC.**
  - 0.5 mm pitch, 0.3 mm pads, 0.2 mm gaps, and **no solder-mask web**: DRC flags 9 `solder_mask_bridge` items, J3.1–10.
  - Drag-solder with generous flux, then wick.
  - Keep the actuator closed and the tip time short; ZIF plastic is heat-sensitive.
  - J3.1/.10 are solid on a pour (N8). Heat them last, briefly, so S1/S8 don't bridge.
- **N10. NOTE: USB-C J2 (GCT USB4125-GF-A, 6-pin, top-mount).**
  - SMD pads 0.70–0.80 mm wide with gaps of 0.29–0.45 mm. The iron is fine.
  - The 4 shell legs are PTH slots, 0.6 × 1.2 mm in 1.1 × 1.7 mm pads, GND with thermals on In1.
  - Solder the SMD row first, with the part held flat. Then do the legs from **B**, filling the slots. They are the mechanical anchor.
  - U1 (B) is between the legs at (134.66, 81.69), 4 mm from each. Solder U1 before the J2 legs, or be careful.
- **N11. NOTE: J5 shunt and height.**
  - The header is 1.27 mm SMD with alternating tails. Pads are 3.0 × 0.65 mm, easy.
  - Neighbours are low: J3 about 1–2 mm tall at ≥ 3.5 mm away; U13 and 0603s at ≥ 3 mm. No clearance conflict for the shunt (M50-2000005; the height with handle is UNVERIFIED).
  - The H3 screw-head/nut keep-out (Ø ≤ 4.6 mm at (124, 118) → edge at y ≈ 115.7) clears J5's pad-2 tail end (y 114.98) by about 0.7 mm. Use a pan-head M2, not a large washer.
- **N12. NOTE: J7 vs C48.**
  - Their courtyards overlap (DRC). The J7 body edge is at x ≈ 157.4, and C48's pads end at x 157.3 and y ≥ 91.88, outside the body.
  - Solder **C48 before J7**; afterwards the iron can't reach C48's +x pad.
- **N13. NOTE: mask-defined pads.**
  - `pad_to_mask_clearance 0` globally. Only the Opa4 pads carry a local 0.102 mm margin.
  - Mask registration (fab tolerance, UNVERIFIED; the fab-capability reviewer covers this) can creep onto U4's 0.25 mm and J3's 0.3 mm pads and shrink the wettable copper.
  - JLC typically applies its own expansion. Confirm, or set about 0.05 mm.
- **Stencil or not.**
  - **No stencil is needed** for iron assembly.
  - A stencil plus hotplate reflow of F is **not recommended** here:
    - 62 unfilled via-in-pads would wick paste and leave starved joints;
    - B also has parts, so F must be reflowed first, and then B hand-soldered with F's tall connectors in the way.
  - To reflow, order **POFV** (epoxy-filled and capped vias). At JLC this is free only for 6–20 layers and charged on 4-layer ([JLCPCB blog](https://jlcpcb.com/blog/Free-Via-in-Pad-on-6-20-Layer-PCBs-with-POFV)).
  - Per part type:

    | Part | Method |
    |---|---|
    | 0603/0805, SOT, SSOP, TQFP, headers, USB-C | iron |
    | U4 | hot air with syringe paste, or iron after S6 |
    | J3 | iron drag, or hot air at low airflow |

### Hand-assembly plan
- **Tools:**
  - temperature-controlled iron with a 1.2 mm bevel or hoof tip and a 0.4 mm conical tip;
  - hot air (for U4);
  - no-clean or water-soluble flux gel;
  - 0.3–0.5 mm solder, 0.8–1.5 mm wick, fine tweezers;
  - 10× loupe or microscope;
  - board holder, preheater optional;
  - IPA for flux clean-up. DESIGN §9.2 notes that leakage at 1 MΩ matters, so clean thoroughly.
- **Order:**
  1. **U4** on the bare board. It has the most room.
     - Hot air: flux, a thin paste bead on the EP (syringe), tin the pins, then 300–330 °C at low airflow.
     - Or the iron-plus-back-via method after S6.
     - Check for bridges and that pins 3/4/6/11 are not shorted to the EP.
  2. **U2** TQFP: tack, drag, wick. Inspect all 32 leads.
  3. **U6, U11, U12, U13** (check pin 1 against the chip dimple and the S1 dot), then **U3, U7**.
  4. **J3 FFC**: fine pitch, while the surroundings are still open.
  5. F passives:
     - charge-pump cluster, using the ibom because of the S2 labels;
     - divider block, one value/reel at a time: 1.1k, then 1k, then 10n;
     - **C48 before J7**;
     - U2 surroundings, then the AFE Rf/Cf pairs;
     - D1, D2, FB1, FB2.
  6. **J2 SMD pins** (F).
  7. Flip to **B** in a holder, supporting it on the flat F areas; no tall F parts are fitted yet except J2.
     - **U1** first.
     - Then R3/R4/R11/R12, C11/C22/C27/C28, C23–C26.
     - Then the diff-amp block **strictly from the ibom**: all 4.99k first, then all 10k (see S3).
  8. **J2 shell legs** from B.
  9. Last: **J5**, then **J7**. These are the tallest parts.
  10. Wash, bake dry, inspect. Do the §9.2 resistance-to-GND checks before power.

---

## 5. Testability and bring-up access

- **Existing test points:** TP1 +5V (123.75, 97.90), TP2 +2V5 (123.53, 94.17), TP3 −2V5 (125.01, 83.77), TP5 GND (125.01, 87.52).
  - All four are **bare Ø1.5 mm SMD pads**: fine for a DMM tip, not clip-able.
- **Missing:** +3.3VA, +3V3, CLKIN, DRDY, Vout, AIN (DESIGN §9 #15). Workable substitutes:

  | Signal | Where to probe |
  |---|---|
  | +3.3VA | C37.1 (168.60, 107.83), FB1.1 (170.51, 109.00) or C61.1 |
  | +3V3 | FB1.2 (172.09, 109.00), C36.1 or C58.1 |
  | MCU_CLK | J7 pin 2, or U3.2 |
  | CLKIN (after the buffer) | R80.1 (172.23, 100.85) |
  | DRDY | R82.2 (174.13, 104.20) or J7 pin 9 |
  | Vout_n | the R(61+2k) pads on the Vout side, on the outer (−x or +x) end of each divider row (`div.png`) |
  | AINkP | the C(48+k) pad that is not on GND |

  All of these are 0603 pads at 1.46 mm pitch: probe with a sharp tip only.
- **N14. NOTE: scope ground.**
  - TP5 cannot hold a ground clip.
  - **H1–H4 are plated GND pads** (Ø3.8 mm, 2.2 mm hole). H1 at (124, 79) is next to TP3/TP5; H4 at (174, 118) is next to U2/FB1. They make good ground-clip points while no screw is fitted, or use a ring lug under the screw.
  - For the AIN/CLK measurements, a probe spring-ground needs a GND pad within about 5 mm. The C48–C55 GND pads work for that.
- **Optional additions** (a copper ECO; re-run DRC):
  - **Bottom-right void** (x 160–172, y 113–120), which is free on F:
    - a THT GND loop (for example Keystone 5001/5011 in a 1.0 mm hole);
    - +3.3VA and +3V3 pads, fed by short stubs from C37.1 and FB1.2.
  - Swap **TP5** for a THT loop, for the charge-pump ripple measurement (§9.2 step 3).
  - If you skip the ECO, the substitutes above are adequate for bring-up.

---

## 6. Documentation and ordering outputs

### Conditional BLOCKER: the outputs in the repo are stale and incomplete. Never upload them.
- They date from 2026-09-16: `PCB v2-*.gbr`, `PCB v2-job.gbrjob`, `PCB v2.csv`, `distinta PCB v3-TESI.xlsx`, `bom/ibom.html`.
- The set has **no In1_Cu / In2_Cu Gerbers and no drill file at all**, so it describes a 2-layer board with no holes.
- **Regenerate from the current board:**
  - Gerbers: F/B Cu, **In1/In2 Cu**, F/B Mask, F/B Silk, Edge.Cuts, and F/B Paste if you want a stencil. Use Protel extensions if the fab wants them.
  - **Drill:** `kicad-cli pcb export drill`, Excellon, separate PTH/NPTH files, with a map.
  - The Gerber job file.
  - The **position file** (F and B), useful as a checklist even for hand assembly.
- **For hand assembly, produce these too:**
  - **InteractiveHtmlBom**, with F/B views and groups by value. This is the key tool given S2/S3 and the 0.5 mm text.
  - An **assembly PDF**, F and B-mirrored: Fab + Silk + Edge, with values on Fab.
  - A fresh BOM CSV. Mark TP1/2/3/5 `exclude_from_bom`, as DESIGN §7 #6 says.
- **N15. NOTE: fab order notes** (paste into the order remarks):
  - 4 layers, 1.6 mm, **custom stackup: 0.1 mm prepreg F.Cu–In1 and In2–B.Cu** (the stackup reviewer owns the details);
  - 1 oz outer / 0.5 oz inner copper, or as the fab's stackup specifies;
  - **ENIG** finish. The file says `copper_finish "None"`. ENIG is flat for the 0.5 mm-pitch U4, U2 and J3, and hand-friendly;
  - **green** mask (best contrast for inspecting fine-pitch joints), **white** silk;
  - "**Vias tented both sides**" (the file has `tenting front yes / back yes`);
  - "**Via-in-pad present, NOT filled/capped, hand assembly, accepted**", so that JLC's engineering query doesn't stall the order;
  - order-number location or removal (S5);
  - "Silkscreen text down to 0.5 mm is intentional, print best effort", or fix S4 and N1 first.

---

## Summary of the tags

- **BLOCKER (conditional):** stale Gerbers. They have no inner layers and no drill; regenerate everything before ordering.
- **SHOULD-FIX:**
  - **S1:** Opa4 pin-1 notch hidden under the body, and a stray bar at the pin-9/10 corner (U6, U11, U12, U13).
  - **S2:** misplaced refs R6 (on R1), C9 (equidistant with C20), C34 (on FB2), C19 (under R9).
  - **S3:** B diff-amp refs centred on the gap between 4.99k and 10k parts.
  - **S4:** 0.5 mm / 0.12 mm knockout TP and J5 labels, below the JLC 1.0/0.15 and PCBWay 0.8/0.15 minimums.
  - **S5:** no board ID, revision or date, and no title block.
  - **S6 (optional):** add B.Mask to U4's EP so it can be iron-soldered through the vias.
  - **Outputs:** produce the ibom and assembly PDFs.
- **NOTE:** N1–N15 above.
