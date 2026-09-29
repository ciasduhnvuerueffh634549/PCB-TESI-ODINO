# Review #3: PCB manufacturability and physical correctness

Board: `PCB v2.kicad_pcb` at git `c884b04`, branch `riccardo_fix`, working tree clean. Tools: KiCad 10.0.6 `kicad-cli` and the `pcbnew` Python API, run on a copy in `scratchpad/review/pcb3/`. The repo was not touched.

## Verdict

- **The fab will accept the board as drawn.**
  - No geometry is outside the JLCPCB 4-layer standard capabilities.
  - DRC finds 0 clearance, short, hole, edge or annular errors, 0 unconnected items and 0 parity issues.
  - No design-file BLOCKER was found.
- **Two order/process BLOCKERs (both known):**
  1. The stackup must be ordered explicitly.
  2. The stale fabrication outputs in `PCB/` must not be used. They are **2-layer** files.
- **SHOULD-FIX items:**
  - solder-mask dams on the op-amps and J3;
  - six 0.15 mm-drill vias that trigger a surcharge;
  - mounting-hole keep-outs;
  - the starved thermals.

---

## 1. DRC triage

Command, run on the copy: `kicad-cli pcb drc --refill-zones --severity-all --schematic-parity`.
Result: **123 violations, 0 unconnected, 0 parity.**

| Type | # | Severity | Where | Verdict |
|---|---|---|---|---|
| text_height | 63 | warn | The 0.5 mm references in the dense blocks, and 7 `gr_text` | Intentional. Set Board Setup → min text height to 0.5 to clear them. See N1 about legibility. |
| lib_footprint_mismatch | 22 | warn | 0603 R/C in the AFE and diff-amp area: C8, C18, C24, C26–C28, R10, R33–R36, R39–R41, R43, R44, R48, R50, R51, R55–R57 | Cosmetic. I diffed C8 against the library `C_0603_1608Metric`: the pads are identical (0.9×0.95 at ±0.775). The only differences are coordinates at the 1e-6 level and a **missing 3D-model entry**. Waive, or "Update footprints from library" with *keep text* to restore the 3D models. |
| lib_footprint_issues | 5 | warn | U6, U11, U12, U13 (`Opa4`) and J3 (`CONNETTORE 687110182122`) | Their libraries are not in any lib table **(known #12)**. The footprints are embedded, so this has no fab impact. |
| starved_thermal | 21 | **error** | Only the small F.Cu power pours, which use thermal gap 0.2 and spoke 1.0: U4 pins 3/6/9/10/11; C6, C7, C17, C19, C20, C33, C34, C60 and FB2.2; D1.2; J2 A9/B9/A12/B12 | A 1.0 mm spoke is as wide as a 0603 pad, so only one spoke fits. Each pad still gets one 1.0 mm spoke, so the connection is electrically fine. **S4**: make these pours *Solid* (better for the charge-pump loop too), or exclude them. |
| solder_mask_bridge | 9 | **error** | J3 pads 1–10 at (125.39, 101.05…105.55) | Real. See **S1**. |
| courtyards_overlap | 1 | **error** | C48 (156.08, 92.35) against J7 | **Not physical (known #9).** See the note below the table. Add a DRC exclusion. |
| silk_overlap | 2 | warn | R37/U6 at (131.2, 99.1); U13/R47 at (134.0, 106.6) | Silk lines touch each other. Cosmetic; the fab clips them. |

Why the C48/J7 overlap is not physical:
- C48's pads end at x ≤ 157.30, y ≥ 91.87.
- J7's nearest pad (pin 12) starts at x ≥ 158.18, y ≤ 91.28.
- J7's plastic body (F.Fab) ends at y 89.77.
- The overlap comes from J7's large courtyard around its 3.15 mm pads.

Also checked on the copy, beyond what DRC reports:
- **Minimum drilled-hole spacing is 0.33 mm.** These are the same-net +2V5 via row at y 82.07, x 157.4–160.6. That is above JLC's 0.2 mm via hole-to-hole minimum.
- **A test Gerber and drill export ran cleanly.**
  - The job file reports 4 layers, 56.1 × 45.1 mm, 1.6 mm and `Finish: None`.
  - The PTH tools are 0.15, 0.20, 0.30, 0.60 (J2 slots) and 2.20 mm.
  - There are no NPTH holes.

## 2. Fab capability (JLCPCB 4-layer, from the capabilities page as fetched 2026-09-29)

| Item | Design | JLC limit | Status |
|---|---|---|---|
| Track / space | 0.10 min track (321 segments). Every net is in Default, so clearance is 0.20. DRC passes at 0.2. | 0.09 / 0.09 (1 oz) | OK |
| Vias | 205 × 0.6/0.3; 3 × 0.3/0.2 (U4 PGOOD/EN+ at (150.14, 83.28), (152.05, 83.17), (152.72, 86.09)); **6 × 0.25/0.15** (GND next to J3 and J5, see S2) | Min 0.15/0.25; "diameter ≥ hole + 0.1 (0.15 preferred)"; **"0.1 or 0.15 mm hole … will cost more"** | OK, but the 0.15 mm vias cost extra (S2) |
| Via annular ring | 0.05 on the small vias; 0.15 on the 0.6/0.3 vias | Meets the +0.1 diameter rule | OK, at the limit |
| Via hole to track | ≥ 0.25 (0.05 ring + 0.2 clearance) | 0.2 | OK |
| Plated slot (J2 shell) | 0.6 × 1.2 drill, 1.1 × 1.7 pad | Minimum 0.35 slot | OK |
| Copper to edge | Rule 0.2; DRC clean | ≥ 0.2 routed | OK |
| Board thickness | 1.60 (stackup sum) | 1.6 standard | OK |
| Copper | File records 0.035 on all 4 layers | Outer 1 oz; **inner 0.5 oz standard** (1 oz optional) | Order 1 oz outer / 0.5 oz inner. The ±2V5 planes carry < 0.1 A. See N7. |
| Finish | `copper_finish "None"` | HASL / ENIG / OSP | **Choose ENIG** (O2) |
| Solder-mask dam | Board `pad_to_mask_clearance 0`, but **Opa4 and J3 pads carry a local `solder_mask_margin 0.102`** | ≥ 0.10 (green, 1 oz) | **U6/U11/U12/U13 dam 0.05; J3 none** (S1). U2 TQFP and U4 WSON dams are 0.2. |
| Silkscreen | 0.8/0.15 (99 texts); 0.5/0.12 (56); 0.5/0.125 knockout (7); 290 lines at 0.12 | Minimum line 0.15, text 1.0 mm, ratio 1:6 | Below JLC's guidance. JLC prints it, but it will not be legible (N1). |
| Via-in-pad | 62 vias inside SMD pads, all 0.3 drill, left open (list below) | Filled/capped is optional and costs extra | Acceptable for hand soldering (D-23). State it on the order. |

**The 0.1 mm prepreg assumption holds with JLC04161H-3313.**

| JLC 4-layer 1.6 mm stackup | Prepreg (L1–L2 and L3–L4) | Core | Inner Cu |
|---|---|---|---|
| **JLC04161H-7628 (the default, "no requirement")** | **7628 = 0.2104 mm** | 1.065 | 0.0152 |
| **JLC04161H-3313** | **3313 = 0.0994 mm** | 1.265 | 0.0152 |
| JLC04161H-1080 | 1080 = 0.0764 mm | 1.265 | 0.0152 |
| JLC04161H-2116 | 2116 = 0.1164 mm | 1.265 | 0.0152 |

- **Order JLC04161H-3313.** It matches the file's 0.10 / 1.24 / 0.10 to within a few µm.
- **If no stackup is specified, JLC builds on 7628 at 0.21 mm**, twice the design value. The guard and return-path arguments in DESIGN §5.2 then weaken. **(known #2)**
- Whether picking a specified stackup adds cost is UNVERIFIED; check at quote time.
- PCBWay was not checked in detail. Ask for "~0.1 mm prepreg (1080/2116/3313 class) on both outer pairs, ~1.2–1.3 mm core" in the remarks.

**The 62 via-in-pad sites** (all open 0.3 mm holes; the pad mask opening leaves them exposed; solder will wick):
- F.Cu GND:
  - C35, C36, C37, C48, C49, C50, C51, C53, C55, C58, C59;
  - R64, R68, R70, R72, R74, R76, R79;
  - U3.3, and U7.2 (two vias).
- F.Cu Vout2: R65.1.
- B.Cu (under U11/U12):
  - supply decoupling C11, C22–C28;
  - Q1–Q3 and Q5–Q7 at R23, R28, R40, R50, R51, R60;
  - D/Vout pads on R24, R29, R34, R36, R38, R39, R48, R49, R59;
  - GND and D-P pads on R26, R31, R44, R46, R56.
- U4's exposed pad also has 3 open 0.3 mm thermal vias (pad 13).

## 3. Zones

- **In1 GND** (priority 2, clearance 0.2, thermal 0.5/0.5, islands always removed):
  - **It fills as one outline, 2410 mm², with no islands.**
  - I tested every F.Cu track segment every 0.05 mm against the In1 fill. The only F.Cu copper over an In1 void is:
    - track ends of 0.5 mm or less entering their own via's antipad;
    - +2V5 along y 82.07 over its own 5-via row (x 157.4–160.6). This is the only merged-antipad slot in In1, and only DC runs over it.
  - **No SPI/CLK/DRDY, AIN, S- or sensor segment crosses a void.**
  - The plane is solid under the AFE and under the digital strip (render: `pcb3/img/In1.Cu.png`).
- **In2 +2V5** (clearance **0.5**):
  - It fills as one outline with no islands.
  - The 0.5 mm clearance produces large merged voids around the U11/U12 via clusters and U4.
  - B.Cu tracks over those voids are all DC or low-impedance: +3V3 up to 2.2 mm; Vsac and Vout2/6/7 up to about 1.8 mm; SYNC_RESET 0.85 mm. Harmless (N6).
- **B.Cu −2V5** (priority 1, islands always removed):
  - It fills as one outline (1960 mm²).
  - DRC has 0 unconnected items, so every −2V5 pad (C11, C24, C26, C28, U4.6 and so on) is reached. The earlier C24/C26 cut (D-22) is gone.
- **F.Cu:**
  - `GND_AFE_GUARD` (priority 1, *solid* pad connection, islands removed) has 11 pieces, all attached to GND.
  - The 16 small power/GND pours (priorities 3–13, clearance 0.2/0.25, thermal 0.2/1.0) are each a single outline with no islands.
  - The priorities give no unintended overlaps. `GND_SUPPLY_F` (priority 8, F+B) correctly overrides −2V5 (priority 1) on B.Cu under J2.
- **Hand soldering:**
  - The planes use thermal relief (0.5/0.5) on THT: the J2 shell slots show 4 spokes on In1. The mounting holes connect to In1.
  - The GND pads inside `GND_AFE_GUARD` are **solid**: J3.1/10/Z1/Z2, J5.1, U13.5/12 and others. Use a 60 W-class iron. J3's 0.5 mm-pitch GND pins will be the slowest joints.

## 4. Mechanical and assembly

- **Mounting holes H1–H4** at (124, 79), (174, 79), (124, 118), (174, 118):
  - 2.2 mm plated holes with 3.8 mm GND pads; 2.74 mm from the side edges, 2.96 mm from the top/bottom edges.
  - Nothing keeps copper out beyond the pad. See S3.
- **J2 USB-C** (GCT USB4125, KiCad footprint for this part):
  - The footprint's "PCB Edge" line (y 76.00–76.09) coincides with the board edge (76.04–76.06).
  - The receptacle is flush with the edge.
  - The pin map is correct: A5/B5 CC, A9/B9 VBUS, A12/B12 GND, 4 SH to GND.
  - U1 (bottom) sits 3.5 mm or more from the shell stakes. OK.
- **J3 FFC** (Würth 687110182122, datasheet p.1: *WR-FPC SMT ZIF Horizontal **Back Locking**, **Dual Contact**, 0.5 mm*):
  - The land pattern matches the datasheet: 10 pads 0.3 × 0.8 at 0.5 pitch; hold-down pads 0.4 × 0.8; 3.3 mm overall and 4.5 mm pin 1–10.
  - The flip actuator side (fab outline, local y −2.25) faces inboard (x ≈ 126.2).
  - The hold-down / cable-entry face is at x 122.54, **1.28 mm from the left edge** (121.26). The FFC enters from the board edge. **OK.**
  - The actuator opens to 1.76 mm high with nothing tall behind it.
  - With dual contacts, either FFC face works, but **flipping the cable reverses S1…S8** (see N3).
- **J7** (2×6 SMD, rotated −90):
  - Pin 1 (SYNC_RESET) is at (171.38, 84.655), top-right.
  - The silk has a pin-1 corner mark and pin-name labels (SYNC… / CLK…).
  - The header is unshrouded, so the cable can be plugged in reversed or offset (see N4).
  - There is 3.7 mm or more to H2's pad. The IDC socket body clears C48 because the socket sits on the header plastic.
- **J5** (Harwin M50-3630342, 1.27 mm):
  - Pads are 3.0 × 0.65 at ±1.5, i.e. 6.0 mm span; the posts are on y 111.98.
  - The nearest obstacles are J3's courtyard (y ≤ 107.15) and H3's pad (y ≥ 116.1). The M50-2000005 shunt fits.
  - A 180° part rotation fits the land (known).
- **Spacing:**
  - The only courtyard conflict is C48/J7 (not physical).
  - The tightest hand-soldering areas are:
    - the divider block: 0603s at 1.46 mm row pitch, leaving about 0.5 mm between pad rows, with via-in-pad;
    - the B.Cu resistor arrays under U11/U12.
  - Solder the B.Cu passives first, then the top side.
- **Tall parts:**
  - J7 header (Würth 61001221121, the tallest part);
  - J5 plus its shunt;
  - J2 (about 3.2 mm);
  - J3 (0.96 mm, 1.76 mm with the actuator open).
  - Everything else is 0603/0805/SSOP/TQFP.
  - The bottom side has only 0603/0805 parts and SC-70 U1: no tall parts, and the board can lie flat on standoffs.
- **Pin-1 and polarity markings** (checked on a high-resolution F.Silk render):

  | Part | Marking |
  |---|---|
  | U2 | ▼ at the top-left |
  | U3 | ▼ |
  | U4 | ◄ |
  | U7 | ▼ |
  | U6, U11, U12, U13 | Notch at the pin-1/16 end (Opa4 silk). Adequate; a dot would be clearer. |
  | D1 (SMF) | Cathode bar (pad 1 = +5V, correct for the TVS) |
  | D2 | Cathode bracket (pad 1 = GND) |
  | J3 | Pin-1 dot |
  | J7 | Pin-1 corner |

  - There are no polarized capacitors.

## 5. Footprints against package drawings

| Ref | Footprint | Check |
|---|---|---|
| U2 ADS131M08IPBSR | `TQFP-32_5x5mm_P0.5mm` | PBS is TQFP-32, 5 × 5 mm, 0.5 mm pitch, **no thermal pad**. Pads 1.475 × 0.3 at ±3.163; dam 0.2. OK. |
| U4 LM27762DSSR | `WSON-12-1EP_3x2mm_P0.5mm_EP1x2.65_ThermalVias` | The KiCad footprint's own description cites the LM27762 datasheet. The EP (pad 13) is on GND, with 3 × 0.3 mm vias. See N5 for hand soldering. |
| U6, U11–U13 OPA4350EA | `Opa4:SOP63P600X175-16N` | SSOP-16 (DBQ): 0.635 pitch, 6.0 mm span. The pads alternate 0.66/0.61 mm, so the worst positional error is ±0.0125 mm. Negligible. Rows ±2.667; pads 1.6 × 0.356. The pinout matches OPA4350 (4 = V+, 13 = V−, 8/9 NC). Only the **mask margin** is wrong (S1). |
| U7 LP5907MFX-3.3 | SOT-23-5 | 1 IN, 2 GND, 3 EN (to +5V), 4 NC, 5 OUT. Matches. |
| U3 SN74LVC1G17DBVR | SOT-23-5 | 1 NC, 2 A, 3 GND, 4 Y, 5 VCC. Matches. |
| U1 TPD2E2U06DCKR | SOT-323 (SC-70-3) | 1 CC1, 2 CC2, 3 GND. Matches. |
| J2 USB4125-GF-A | KiCad GCT USB4125 6P | Exact part footprint, edge-aligned. Matching the stake length to the 1.6 mm board is UNVERIFIED (the KiCad description says "1 mm stake"). |
| J3 687110182122 | Custom (SnapEDA) | Matches the Würth land pattern (above). |
| J5 M50-3630342 | `PinHeader_1x03_P1.27mm_Vertical_SMD_Pin1Right` | Matches the Harwin land pattern (0.65 × 3.0, 6.0 mm span). |
| J7 61001221121 | KiCad 2×06 SMD | Pads 3.15 × 1.0 at ±2.525, against Würth's 2.3 × 1.0 on an 8.5 mm span. The 7.5 mm tail span lands on the pads (known, DESIGN §7.8). |

---

## Findings

### BLOCKER

**B1. Order the stackup explicitly: JLC04161H-3313 (known #2).**
- **Location:** the order form, and board `setup/stackup`.
- **Evidence:**
  - The file assumes 0.10 mm for F.Cu–In1 and In2–B.Cu.
  - JLC's default or "no requirement" stackup is 7628 at 0.2104 mm.
  - 3313 is 0.0994 mm, with a 1.265 mm core and 0.5 oz inner copper.
- **Impact:** without it, the outer dielectric is 2× thicker. The guard, crosstalk and return-path assumptions (DESIGN §5.2) are off by 2×. The board still functions, but not as designed.
- **Fix:**
  - Select *Specify stackup → JLC04161H-3313*.
  - Optionally update the file's stackup to 0.0994 / 0.0152 / 1.265 / 0.0152 / 0.0994 for the record.

**B2. Do not upload the Gerbers in `PCB/`; they are for a different, 2-layer board (known #3).**
- **Location:** `PCB/PCB v2-*.gbr`, `PCB v2-job.gbrjob`.
- **Evidence:** the repo's job file says `"LayerNumber": 2`, size 53.75 × 40.72 mm, created 2026-08-22.
- **Impact:** uploading them builds the obsolete AD7606-era 2-layer board. Every board would be scrap.
- **Fix:**
  - Delete or move them.
  - Regenerate the outputs (section O1). A test export on the copy produced a correct 4-layer, 56.1 × 45.1 mm set.

### SHOULD-FIX

**S1. Solder-mask dams are missing on J3 and U6/U11/U12/U13.**
- **Location:** J3 pads 1–10 at x 125.389, y 101.05–105.55; every pad of the Opa4 footprints.
- **Evidence:**
  - These pads carry a local `solder_mask_margin 0.102` (SnapEDA default). The board default is 0.
  - J3: pad gap 0.20 − 2 × 0.102 gives a **−0.004 mm** dam, which is the 9 DRC `solder_mask_bridge` errors.
  - Opa4: pad gap 0.254 gives a **0.05 mm** dam, under JLC's 0.10 mm minimum. JLC will remove it, so there is gang relief along every op-amp pin row.
- **Impact:** no mask between the 0.5 mm FFC pins and the 0.635 mm op-amp pins means a higher bridging risk when hand soldering. Sensor-input shorts on J3 would be hard to see.
- **Fix:**
  - In each footprint's properties, set the pad solder-mask expansion to 0 (use the board default), or to 0.025 at most.
  - That gives dams of 0.20 (J3) and 0.25 (Opa4).
  - Five footprints; about 5 minutes.

**S2. Six 0.25/0.15 vias trigger JLC's small-hole surcharge.**
- **Location:** GND at (122.89, 99.50), (125.389, 100.49), (125.39, 106.14), (122.89, 107.13), (124.59, 112.63), (124.04, 115.61).
- **Evidence:** JLC says "0.1mm or 0.15mm hole size … will cost more". These are the only 0.15 mm holes (drill tool T1).
- **Test:** I changed all six to **0.3/0.2** on the copy. A full DRC then gave **no new violations**: the same 21 starved thermals, 1 courtyard overlap and 9 mask bridges.
- **Impact:**
  - As drawn, this is a price surcharge.
  - The ring is 0.05 mm, the absolute minimum.
  - Several board houses do not offer 0.15 mm holes at all.
- **Fix:** set those six vias to 0.3/0.2 (the U4 vias are already 0.3/0.2), then select 0.2/0.3 mm as the minimum via on the order form.

**S3. Mounting holes have no copper keep-out beyond the 3.8 mm pad.**
- **Location:** H1–H4. S_sac also passes near H3.
- **Evidence:**
  - The B.Cu −2V5 pour starts 2.1 mm from every hole centre, i.e. only 0.2 mm outside the GND pad.
  - The S_sac F.Cu track is 2.45 mm from H3's centre at (124, 118).
  - An M2 nut (4.6 mm across corners), a washer (5 mm OD) or a hex standoff overlaps that copper through the mask alone.
- **Impact:** metal GND hardware pressing on or scratching the mask can short −2V5 to GND. On top it can load or short S_sac, the reference sensor input.
- **Fix, either of:**
  - Add rule-area keep-outs (no copper on F.Cu and B.Cu) of radius about 3 mm around H1–H4. Check that S_sac can be nudged away from H3.
  - Use nylon screws, washers and standoffs only, and note this in the assembly notes.

**S4. 21 starved thermals in the charge-pump and USB pours.**
- **Location:** the U4/J2 F.Cu pours (table above).
- **Evidence:** thermal 0.2 gap / 1.0 mm spoke on 0603/0805 and 0.25 mm WSON pads, where only one spoke fits.
- **Impact:** none electrically. It leaves DRC errors, and the switched-capacitor loop gets slightly more inductance than a solid pour.
- **Fix:** set those zones' pad connection to *Solid* (small pours are easy to hand-solder), or exclude the errors in DRC.

**S5. Board identification and JLC order-number placement.**
- **Evidence:** the silkscreen has no board name, revision or date; the only user texts are the 7 TP/J5 labels.
- **Impact:**
  - Revisions cannot be traced.
  - JLC prints its order number at a random free spot, which on this board could land in the dense blocks.
- **Fix:**
  - Add "PVDF e-skin ADC v2 2026-09" in 0.8–1.0 mm text on B.Silk.
  - Add a `JLCJLCJLCJLC` text at a chosen free spot, or pick "specify location" on the order form.

### NOTE

- **N1. Silk legibility (intentional; known #11).**
  - 56 references are 0.5/0.12 mm and 7 labels are 0.5/0.125 mm knockout; 290 footprint silk lines are 0.12 mm.
  - JLC's guidance is 1.0 mm text and 0.15 mm lines.
  - It will print, but expect 0.5 mm text to be marginal and the **0.5 mm knockout TP/J5 labels to fill in**. Consider normal (non-knockout) 0.8 mm text for those 7.
  - Rely on the regenerated iBOM.
- **N2. Library hygiene (known #12).** 22 mismatches are the same pads with a missing 3D model; 5 are unregistered libraries. No fab impact.
- **N3. FFC orientation.**
  - J3 is dual-contact, so a flipped FFC (type A vs type B, or a flex with contacts on the other face) still makes contact but **mirrors the pin order**: S8 ↔ S1 and so on.
  - Pins 1 and 10 are both GND, so nothing is damaged.
  - Document the FFC type (same-side or opposite-side contacts) that matches the sensor flex, and check with a continuity test.
- **N4. J7 is unshrouded (known #10).**
  - The pin-1 corner mark and the SYNC/CLK labels exist.
  - Mark pin 1 on the cable or IDC socket (red stripe to SYNC, at the top-right, x 171.4).
  - Buzz out against the STM32H7 board before first power.
- **N5. U4 (WSON-12 with EP and 3 open vias).**
  - Hand soldering needs hot air or a hotplate. Tin the EP first.
  - The vias wick solder, so add extra solder or feed some through the vias from B.Cu.
  - Inspect pins 1–12 for bridges at 0.5 mm pitch.
- **N6. In2 antipads.**
  - The 0.5 mm clearance on In2 is larger than needed and merges antipads under U11/U12.
  - Only DC or low-impedance B.Cu nets cross them (at most about 2.2 mm).
  - Next revision: use 0.25–0.3.
- **N7. Stackup record.** The file says 1 oz inner copper and a 1.24 mm core. JLC-3313 is 0.5 oz and 1.265 mm. Update the file for the thesis documentation; electrically irrelevant.
- **N8. Top edge.** The top edge is tilted by 0.02 mm (known #18). The fab ignores this.
- **N9. Stale outputs.** `PCB v2.csv`, the `.xlsx` and `bom/ibom.html` are stale (known). Regenerate the iBOM from the current board before assembly.

---

## O1. Outputs to generate (from the GUI Plot/Drill dialogs, or `kicad-cli` on a copy)

1. **Gerber X2:**
   - F.Cu, In1.Cu, In2.Cu, B.Cu;
   - F.Mask, B.Mask;
   - F.Silkscreen, B.Silkscreen (tick *subtract soldermask*);
   - Edge.Cuts.
   - Paste layers only if you order a stencil (not needed for hand soldering).
2. **Gerber job file** (`.gbrjob`). Set `Finish` by first choosing ENIG in Board Setup → Board Stackup → Board finish.
3. **Excellon drill** in mm, PTH and NPTH separate (NPTH is empty), plus a drill map (optional).
4. **Fab notes** (README in the zip): the stackup, "no impedance control", "via-in-pad intentional, leave open", finish and mask colour.
5. **For assembly:** a regenerated iBOM and the Mouser BOM. No pick-and-place file is needed for hand assembly.

Tested on the copy:

```
kicad-cli pcb export gerbers --check-zones -l "F.Cu,In1.Cu,In2.Cu,B.Cu,F.Mask,B.Mask,F.Silkscreen,B.Silkscreen,Edge.Cuts" --subtract-soldermask -o fab/ "PCB v2.kicad_pcb"
kicad-cli pcb export drill --format excellon --excellon-units mm --excellon-separate-th --generate-map --map-format gerberx2 -o fab/ "PCB v2.kicad_pcb"
```

## O2. Fab order settings (JLCPCB)

```
Base material ......... FR-4 (TG155 ok)
Layers ................ 4
Dimensions ............ 56.1 x 45.1 mm (auto from Gerbers)
Thickness ............. 1.6 mm
Specify stackup ....... YES -> JLC04161H-3313  (L1-L2 & L3-L4 prepreg 3313 = 0.0994 mm, core 1.265 mm)
Impedance control ..... none required (if the form ties stackup choice to "impedance: yes", pick yes, no targets)
Outer copper .......... 1 oz
Inner copper .......... 0.5 oz
Min via hole/diam ..... 0.2/0.3 mm  (0.15/0.25 if S2 is NOT applied -> surcharge)
Surface finish ........ ENIG (flat pads for 0.5 mm-pitch TQFP-32, WSON-12 and the FFC; better for hand soldering than HASL)
Solder mask ........... Green (0.1 mm dam minimum; other colours need more)
Silkscreen ............ White
Via covering .......... Tented (NOT "plugged" and NOT epoxy/copper filled). Vias inside SMD pads stay open by design.
Remove order number ... Specify a location (JLCJLCJLCJLC text) or pay to remove
Castellated / edge plating / gold fingers ... No
Remark ................ "Via-in-pad intentional (hand assembly): leave open, no filling/capping.
                         Stackup must be JLC04161H-3313 (0.1 mm outer dielectric)."
```

For PCBWay or others: 4 layers, 1.6 mm; outer prepreg about 0.1 mm (1080, 2116 or 3313 class) on both sides; core 1.2–1.3 mm; 1 oz outer / 0.5 oz inner; ENIG; tented vias; via-in-pad left open; minimum drill 0.2 mm.

---

## Summary

- **BLOCKER B1 (known):** order the stackup JLC04161H-3313 (0.0994 mm prepreg). JLC's default 7628 is 0.21 mm, double the design's 0.1 mm assumption.
- **BLOCKER B2 (known):** the Gerbers and job file in `PCB/` are for an obsolete 2-layer, 53.75 × 40.72 mm board. Regenerate the outputs; a test export on a copy is correct (4 layers, 56.1 × 45.1 mm).
- **No design-file blocker.**
  - DRC: 0 clearance, short, hole or unconnected errors; 0 parity issues.
  - All geometry is within JLC 4-layer capabilities.
  - The In1 GND plane is solid under every SPI, CLK, AIN and sensor trace.
  - The ±2V5 pours have no islands and reach every pad.
  - J2 is flush with the edge. J3 cable entry faces the edge. J7 pin 1 is marked. Footprints match the MPNs.
- **S1:** set the J3 and OPA4350 footprint pad mask margin from 0.102 to 0. The dams are currently −0.004 and 0.05 mm, against JLC's 0.1 mm minimum.
- **S2:** change the six 0.25/0.15 GND vias near J3/J5 to 0.3/0.2. This is DRC-clean (tested on a copy) and removes the small-hole surcharge.
- **S3:** add about 6 mm-diameter copper keep-outs at H1–H4, or use nylon hardware. The −2V5 pour sits 0.2 mm outside the GND hole pads, and S_sac runs 2.45 mm from H3.
- **S4 and S5:** set the small power pours to solid connection (clears the 21 starved thermals); add a board name/revision and a JLC order-number spot.
- **Order settings:** ENIG, green mask, tented vias, via-in-pad left open, 1 oz outer / 0.5 oz inner copper.
