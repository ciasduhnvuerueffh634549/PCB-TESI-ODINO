# Review 1: logical circuit vs requirements and scope

**Scope.** Does the circuit as drawn do what the board is for?

**Method.**
- Exported a netlist with kicad-cli 10.0.6 from a scratch copy of the project (`review/requirements/net.net`). Every stage below was checked net by net against it.
- Datasheets checked, downloaded from ti.com:
  - ADS131M08 (SBAS950B);
  - OPA4350 (SBOS099D);
  - LM27762;
  - SN74LVC1G17.
- Requirements come from DESIGN.md, the CHANGELOG, the FOGLIO1/FOGLIO2 text notes, the git log and the user turns in the transcripts.

**Verdict.** The analog and digital circuit is logically correct and matches its stated intent. **I found no hardware blocker.**

The real gaps are these:
1. The firmware plan (1 MHz SCLK) cannot meet the 10 kSPS requirement.
2. Several requirements the gain sizing depends on were never defined: signal amplitude, bandwidth and sensor capacitance. With 1 MΩ ∥ 15 pF, slow or quasi-static pressure is essentially not measurable.
3. The sensor inputs have no protection.

---

## 1. Requirements table

| # | Requirement | Source | Met? | Evidence |
|---|---|---|---|---|
| R1 | 7 measurement channels + 1 reference ("sacrificial") channel, all digitised | DESIGN §1; FOGLIO1 note "CH7 digitises the sacrificial-sensor output Vsac"; D-3 | **Yes** | Netlist: S1–S7 → U6-A/B/C/D, U13-C/D/A → Q1–Q7 → U11/U12 → Vout1–7 → AIN0–6P; S_sac → U13-B → Vsac → R75/R76/C55 → AIN7P (U2.12). |
| R2 | Sensor = PVDF film, charge output, one electrode per taxel, common electrode to GND | DESIGN §1, FOGLIO2 heading "CHARGE AMPLIFIERS" | **Yes** (topology) | Inverting stage, −IN = sensor, +IN = GND (U6/U13 pins 3/5/12/14 on GND). J3 pins 1, 10, Z1, Z2 on GND. |
| R3 | Stage-1 response: "under the pole, flat band, transimpedance" (designer, transcript 2026-09-28) | Transcript user turn; D-17 | **Yes** as specified | Rf 1 MΩ ∥ Cf 15 pF → f_c = 10.61 kHz, above the 2.6 kHz digital bandwidth, so the output ∝ dQ/dt ∝ dF/dt everywhere in band. |
| R4 | Signal bandwidth "0.1 Hz to a few kHz" (tactile band) | DESIGN §2.2 (agent-stated, not a designer requirement) | **Partly** | Upper end: sinc3 −3 dB ≈ 0.262·f_DATA ≈ 2.6 kHz, fine. Lower end: in current mode the charge→V gain at 0.1 Hz is 2π·0.1·1 MΩ = 0.63 µV/pC, so quasi-static pressure is not measurable (F-2). |
| R5 | Signal amplitude / full scale | **Not defined anywhere.** The assistant asked on 2026-09-28 ("What signal amplitudes do you expect?"); no answer in the transcripts. DESIGN §2.6 "open question". | **Unknown** | Hardware full scale is ±1.22 µA of (i_n − i_sac) (diff-amp clip) ≈ ±1.26 µA (ADC FS). Expected: 1 N in 10 ms at d33 ≈ 25 pC/N → 2.5 nA → 2.4 mV at the AIN pin, 0.2 % of FS (estimate). See F-3. |
| R6 | Sample rate 10 kSPS on all 8 channels, simultaneous | Transcript user turn: "sample at maximum 10kSPS all eight channels"; D-7; FOGLIO1 J7 note | **Yes (HW)**, **No with the stated 1 MHz SCLK** | 5.12 MHz / 2 / 256 = 10.000 kSPS (DS §8.3.6: f_MOD = f_CLKIN/2, OSR = f_MOD/f_DATA). 5.12 MHz is within the HR-mode range of 0.3–8.4 MHz (gain 1–2) and 0.3–8.2 MHz (gain > 2) (DS §6.3). SPI throughput: see F-1. |
| R7 | Resolution: 24-bit ADC, "16-bit data" (designer) vs "32-bit sign-extended" (sheet note) | Transcript; FOGLIO1 note; CHANGELOG l.143 | **Partly** | 24-bit, ENOB 17.8 at OSR 256 / gain 1 (DS Table 7-2); noise 10.68 µVrms (Table 7-1). 16-bit words at gain 1 cost about 3 dB (F-3). |
| R8 | Sacrificial-sensor rejection: Vout_n = 2·(Vsac − Q_n), S8 or GND selectable | FOGLIO2 note; DESIGN §2.3; J5 | **Yes** | All 7 diff amps checked: Rin 4.99 k Q_n→D_nN, Rfb 10 k D_nN→Vout_n, Rp 4.99 k Vsac→D_nP, Rg 10 k D_nP→GND (e.g. R23/R24/R25/R26 for CH1; R60/R59/R55/R56 for CH7). Gain 2.004, polarity as documented. J5: 1 = GND, 2 = S_sac, 3 = S8. |
| R9 | Host = STM32H7 at 3.3 V logic, supplies CLKIN, SPI, CS, SYNC/RESET, reads DRDY | FOGLIO1 J7 note; D-6 | **Yes** | J7: 1 SYNC_RESET, 2 MCU_CLK, 4 MOSI, 5 CS, 8 MISO, 9 DRDY, 10 SCLK, 3/6/7/11/12 GND. Levels OK: U2 V_IH = 0.8·DVDD = 2.64 V; H7 V_OH ≥ 2.9 V. SPI mode 1 matches DS §8.5.1.2. |
| R10 | Power from a single USB-C 5 V (power-only); analog ±2.5 V; ADC 3.3 V | FOGLIO1 notes; D-5, D-26, D-30 | **Yes** | R1/R6 = 130 k/120 k → 1.2·(250/120) = 2.50 V. R2/R9 = 105 k/100 k → −1.22·(205/100) = −2.50 V (LM27762 DS eq. 1/3, V_FB+ 1.2 V, V_FB− −1.22 V). U7 LP5907-3.3 → +3.3VA → FB1 → +3V3. OPA4350 at 5 V total is within 2.7–5.5 V. |
| R11 | Sensor connection: FFC to the sensor flex | DESIGN §6 | **Partly** | J3 = Würth 687110182122, 10 p, 0.5 mm, ZIF, **dual contact**. The flex pinout and the mating FFC are not documented or in the BOM (F-9). |
| R12 | ADC input never exceeds its limits, even with an op-amp saturated | D-4 | **Yes** | ±2.5 V · 0.476 = ±1.19 V worst case. DS §6.3: absolute input AGND−1.3 V to AVDD (gain 1–4) / AVDD−1.8 = 1.5 V (gain ≥ 8). Abs max AGND−1.6 V. |
| R13 | Hand-solderable, bring-up-friendly | Transcript (via-in-pad decision) | **Partly** | 4 rail TPs only; no TP on Vsac/Q/Vout/AIN/CLK (F-7). |

---

## 2. Stage-by-stage check (logic and numbers)

**Stage 1 (U6, U13).**
- Topology and values are correct for the stated transimpedance intent.
- In band, Q_n = −R_f·i_n.
- Stability is fine for any sensor capacitance: the high-frequency noise gain is 1 + Cs/Cf, a flat β, and the OPA4350 is unity-gain stable.
- Worst-case offset is ±0.5 mV (V_OS max at 25 °C, DS §6.6).
- The +IN pins at 0 V sit 2.5 V below V+. That is at the edge of the OPA4350's worst-case input-pair transition region, (V+)−2.4 V to (V+)−1.2 V (DS §7.3), but outside it. OK.

**Stage 2 (U11, U12).**
- Polarity and ratios verified on all 7 channels (R8).
- Resistor-limited worst-case CMRR ≈ (1+G)/(4·0.1 %) ≈ 57 dB.
- The diff amp clips at |Vsac − Q_n| = 1.22 V, half the stage-1 swing. Acceptable, because the common-mode part is removed before the gain.
- Vsac load: 7 × 14.99 k ∥ 2.1 k = 1.06 kΩ, about 2.3 mA. That is within the OPA4350's ±40 mA and its ≤ 200 mV-from-rail swing at 1 kΩ.
- The diff-amp +IN sits at 0.667·Vsac. For Vsac > +0.1…+1.3 V it passes through the input-pair transition region, so CMRR degrades a little on large positive reference excursions (NOTE, F-12).

**Divider and anti-alias.**
- Ratio 0.4757. The ADC input impedance is 330 kΩ·4.096 MHz/f_MOD = **528 kΩ** at f_MOD = 2.56 MHz (DS eq. 4), not 330 kΩ as DESIGN §2.4 says; the ratio changes by 0.04 %, which is trivial.
- 524 Ω ∥ 10 nF C0G → 30.4 kHz, which is −38.5 dB at f_MOD = 2.56 MHz.
- TI states that a single-order RC is sufficient and recommends 1 k/10 nF/1 k for CLKIN 2–8.2 MHz (DS §9.1.2). Compliant.

**ADC scaling.**
- End-to-end: 0.953 V per µA of (i_n − i_sac) at the AIN pin.
- A saturated op-amp lands at 97–99 % of FS. That is good use of the range, but gives very little resolution at the expected amplitudes (F-3).

**Clocking.**
- MCU_CLK (J7.2) → R79 100 k pull-down → U3 (74LVC1G17, I_off partial-power-down, DS §8) → R80 33 Ω → CLKIN (U2.23).
- XTAL2 is NC.
- f_DATA = 10 kSPS at OSR 256. Duty cycle must be 40–60 % (DS §6.3).
- Default after reset: OSR = 1024, PWR = HR, XTAL_DIS = 0, WLENGTH = 24-bit (CLOCK reset FF0Eh, MODE reset 0510h).
- XTAL_DIS = 1 is a power-saving recommendation (DS §8.3.5), not a functional requirement as DESIGN §2.5 implies.

**Reset / SYNC.**
- R78 10 k pulls SYNC/RESET up, so the ADC is never held in reset with J7 unplugged.
- A reset needs ≥ 2048 t_CLKIN = 400 µs with CLKIN running. A sync pulse must be 1–2047 t_CLKIN (DS §6.6).
- t_POR = 250 µs. DRDY going low→high marks SPI-ready (DS §8.4.1).
- R77 pulls CS up. Logic is correct.

---

## 3. Findings

### BLOCKER
None found in the logical circuit.

### SHOULD-FIX

**F-1: The planned 1 MHz SCLK cannot sustain 10 kSPS; "~8 MHz" in the sheet note is not synchronous with 5.12 MHz. (known, DESIGN §9.1 item 1)**
- **Location:** FOGLIO1 J7 note ("~8 MHz"); transcript ("we are thinking about a 1MHz SCLK"; "SPI goes at 1MHz"). This is a firmware decision that must be settled before firmware is written; no hardware change is needed.
- **Evidence (confirmed against the datasheet):**
  - A data frame is always 10 words: response + 8 channels + CRC (DS §8.5.1.7).
  - "The SPI frame can be shortened … only … if the ADCs are disabled" (DS §8.5.1.11), so no shortened frames while converting.
  - Word length is 16/24/32 bits (§8.5.1.8), giving 160/240/320 SCLK per frame. At 1 MHz that is 160/240/320 µs against a 100 µs period.
  - The FIFO is only 2 deep. Missed samples need a SYNC strobe or a double read (§8.5.1.9.1).
  - DRDY pulses are blocked if a conversion completes during a read (§8.5.1.5), so frames must finish well inside the period.
- **Impact:** at 1 MHz the maximum usable configuration is OSR 512 → 5 kSPS with 16-bit words (160 of 200 µs, which violates the "avoid reading across conversion completion" advice), or OSR 1024 → 2.5 kSPS with 24-bit words. The 10 kSPS requirement is not met.
- **Clock-source issue:** D-6 requires SCLK and CLKIN to come from the same PLL. From 40.96 MHz (or VCO 409.6 MHz), the H7's power-of-two SPI prescalers give 5.12/10.24/2.56 MHz, not 8 MHz.
- **Fix:**
  - Use **SCLK = 5.12 MHz**, i.e. SPI kernel 10.24 MHz ÷ 2, or 40.96 MHz ÷ 8, from the PLL2 that also feeds MCO2. A 24-bit frame then takes 46.9 µs and a 32-bit frame 62.5 µs, within 100 µs.
  - Or use 2.56 MHz with 16-bit words: 62.5 µs.
  - Maximum SCLK is 25 MHz (t_c(SC) ≥ 40 ns).
  - Update the FOGLIO1 note from "~8 MHz" to "5.12 MHz (≥ 2.56 MHz)".

**F-2: The 1 MΩ ∥ 15 pF first stage cannot measure slow or quasi-static pressure. The stated "0.1 Hz – few kHz" band is not achievable. (known, D-17: designer accepted)**
- **Location:** R10, R22, R27, R32, R37, R42, R47, R57 (1 M) and C8, C10, C12–C16, C18 (15 p).
- **Evidence:**
  - Charge-to-voltage gain in band = 2π·f·R_f. At 10 Hz that is 9.4·10⁻⁴ of the charge-amp gain (1/Cf), i.e. −60 dB. At 0.1 Hz it is −100 dB.
  - The op-amp offset (±0.5 mV max) equals 0.5 nA of input current. At 25 pC/N that is about 20 N/s of apparent dF/dt. Offset drift (4 µV/°C) equals about 4 pA/°C.
  - A 1 N press over 1 s gives 25 pA → 24 µV at the AIN pin. At gain 1 the per-sample noise floor is about 14.7 pA rms: the Rf pair gives 9.5 pA over a 2.7 kHz ENBW, and the ADC 11.2 pA (computed).
  - So after the DC offset is removed (which digital integration requires), anything slower than roughly 1 Hz is buried in offset drift and noise.
  - Dynamic events (taps, slip, vibration, 1 N in 10 ms → 2.5 nA → SNR about 170 per sample) are fine.
- **Impact:** if the thesis needs force or pressure profiles, not only contact events and dynamics, the first stage does not deliver them.
- **Fix (BOM-only, no re-layout):**
  - Confirm the requirement with the supervisor now.
  - If force is wanted, change Rf/Cf on the same 0603 pads. Session Option A was **1 GΩ / 220 pF** (0.7 Hz corner, about 110 mV/N, about ±11 N FS).
  - A cheaper hedge: order a second set of values, e.g. **10 MΩ + 1 nF C0G** (16 Hz corner, 1 mV/pC ≈ 25 mV/N), to populate one board as a true charge amp.
  - All 8 channels, including R10/C8, must be changed together to keep the Vsac subtraction matched.
  - Higher Rf increases sensitivity to leakage: wash the board.

**F-3: The expected amplitude is undefined, and at gain 1 the chain uses about 0.2 % of the ADC range. The designer's 16-bit word plan then makes quantisation noise comparable to the ADC noise. (partly known, DESIGN §2.6 and §10.2)**
- **Evidence:**
  - FS = ±1.26 µA of differential current at the ADC.
  - 1 N in 10 ms → 2.4 mV at AIN, which is 65 LSB in 16-bit mode (1 LSB = 36.6 µV ≈ 38 pA).
  - 16-bit quantisation noise is 11.1 pA rms against 11.2 pA of ADC noise at gain 1: about +3 dB total.
  - The PGA's absolute input range for gain ≥ 8 is AGND−1.3 V to AVDD−1.8 V = 1.5 V (DS §6.3), so raising the gain is electrically safe even for full-swing inputs.
- **Impact:** resolution is wasted, and the gain choice is a guess until the signal size is known.
- **Fix (firmware):**
  - Default to 24-bit or 32-bit sign-extended words. This needs SCLK ≥ 2.4/3.2 MHz (F-1).
  - Pick the PGA gain per channel after first measurements. Gain 4 (±300 mV FS) already makes 16-bit quantisation negligible. Gain 32 (±37.5 mV) gives a noise floor of about 9.8 pA but clips above about 39 nA (e.g. 1 N in < 0.6 ms).
  - Keep Vsac (AIN7) at gain 1, since it has half the gain of the other channels.

**F-4: No protection on the sensor inputs. The OPA4350 is rated only ±1 kV HBM, on a surface people touch. (known, DESIGN §9.1 item 7)**
- **Location:** nets S1–S7 and S8/S_sac run J3 → Rf/Cf → U6/U13 −IN directly (netlist). J5 is also exposed.
- **Evidence:**
  - OPA4350 DS §6.2: HBM ±1000 V.
  - Abs max input (V−)−0.3 to (V+)+0.3 V, 10 mA (DS §6.1).
  - PVDF produces kV-level open-circuit spikes. In the stage-1 saturated case, V_sensor = Q/Cs (e.g. 250 pC into 100 pF gives 2.5 V).
- **Impact:** a finger-ESD event or a hard impact can destroy an input or degrade its leakage, which matters at 1 MΩ and more with a higher Rf. A dead op-amp input takes out a whole quad (4 channels).
- **Fix:**
  - Next revision: 1 kΩ in series between each J3 pin and its S node, before the Rf/Cf junction. This adds a pole at 160 kHz with 1 nF, and its noise is negligible against Rf. Optionally add a low-leakage clamp pair to GND.
  - This board: it cannot be added without re-layout inside the guard. Accept it as a known limitation:
    - make sure the flex's outer (touched) electrode is the GND/common electrode;
    - handle the flex with ESD precautions;
    - buy spare OPA4350s.

**F-5: MCU-first power-up can back-power the ADC through its digital inputs. There is no hardware current limit. (partly known, DESIGN §9.2 step 5, as a precaution only)**
- **Location:** ADC_CS (J7.5 → U2.17, R77 to +3V3), ADC_SYNC_RESET (J7.1 → U2.16, R78), ADC_SCLK (J7.10 → U2.19), ADC_MOSI (J7.4 → U2.21). None has a series resistor on this board.
- **Evidence:**
  - U2 digital inputs have an abs max of DVDD+0.3 V and 10 mA per pin (DS §6.1).
  - The H7 idles CS and SYNC high. When the ADC board is unpowered (USB unplugged), that current flows through U2's ESD diodes into +3V3/+3.3VA, limited only by the H7 driver.
  - The sheet's 33 Ω at the H7 end limits it to about 90 mA, which is not enough.
  - MCU_CLK is safe: U3 has I_off (SN74LVC1G17 DS §8).
  - DRDY/MISO driving an unpowered H7 is fine on H7 FT pins (unverified per pin).
- **Impact:** likely in a lab where the ST-Link powers the H7 first: latent damage or odd start-up.
- **Fix (MCU side, no board change):**
  - Change the H7-side series resistors to **1 kΩ on CS and SYNC_RESET** and **220–330 Ω on SCLK, MOSI and MCU_CLK** (with ~10 pF that is a 3 ns RC, fine at 5–10 MHz). This keeps the current ≤ 10 mA.
  - Firmware: keep these pins Hi-Z, with an internal pull-down on DRDY, until DRDY reads high. U2 drives DRDY high after t_POR, so DRDY high means the ADC board is powered.

### NOTE

**F-6: Offset and self-test are available. Use them.**
- ADC side: CHn_CFG.MUXn can connect the channel to AGND or apply ±DC test signals (DS §8.3.2, §8.3.9). Global-chop cuts the ADC offset from ±240 µV to ±32 µV (DS §6.5).
- Digital DC-block filter: DCBLOCK in THRSHLD_LSB, 9.7 mHz–181 Hz corner at 4 kSPS (Table 8-4); corners scale with the data rate.
- Front-end side: J5 on 1–2 grounds S_sac, so Vout_n = −2·Q_n and each stage-1 offset is measured directly.
- J5 open is benign: S_sac is a virtual-ground node closed by Rf∥Cf, not really "floating" as DESIGN §6 says. It is equivalent to a zero-capacitance sensor plus the stub.
- Calibrate the per-channel offset before any digital integration (F-2).

**F-7: Bring-up access is limited.**
- There are TPs only for +5V, ±2V5 and GND (TP1/2/3/5).
- Probe the following on pads:
  - Vsac on R75 pad 1 or C8 pad 2;
  - Q_n on the Rf/Cf output pads (F.Cu);
  - Vout_n on the 1.1 k divider tops (F.Cu);
  - AIN on C48–C55;
  - CLKIN on R80;
  - DRDY on R82.
- (known, DESIGN §9.1 item 15.) Adequate. Consider TPs for Vsac, +3.3VA and CLKIN in the next revision.

**F-8: Running without the H7.**
- The ADC cannot run without an external CLKIN: there is no crystal and no XO footprint.
- J7 is a plain 3.3 V header, though. For debug, a USB-SPI bridge (e.g. FT232H/MCP2210) plus a 5.12 MHz function generator (square wave, 0–3.3 V, 40–60 % duty) into J7.2 can exercise it.
- The analog chain is fully observable with a scope and no MCU.
- A DNP XO footprint could be added in the next revision.

**F-9: The FFC orientation and pinout must be defined against the sensor flex. (partly known: the FFC is not in the BOM)**
- J3 is a dual-contact ZIF (Würth 687110182122), so the flex fits either face up or face down.
- Flipping the flex mirrors pins 2↔9: S1 ↔ S8. The reference electrode would then land on the S1 amplifier and a taxel on the reference.
- The GND pins at 1 and 10 are symmetric, so nothing prevents this.
- Document the flex pinout and mark pin 1 on the flex and the silkscreen.

**F-10: The Vsac subtraction is a fixed 1:1 in analog.**
- If the reference electrode differs in area or capacitance from the taxels, the cancellation is imperfect, and it adds √2 of Rf noise to every channel.
- Because AIN7 digitises Vsac, the raw channel can be rebuilt in firmware as Q_n = Vsac − Vout_n/2.004 (valid while unclipped) and a weighted subtraction k_n·Vsac applied instead.
- This is good for the thesis: it lets you compare analog and digital rejection.

**F-11: Power-good and sequencing.**
- LM27762 PGOOD (R11 pull-up) is unused and not on J7 (known, item 14).
- The analog-to-ADC input path is self-limiting: with AVDD = 0 and AIN at +1.19 V, the diode current is (1.19−0.3) V / 524 Ω ≈ 1.7 mA, below the 10 mA abs max. No sequencing issue on the analog side.

**F-12: Diff-amp common mode in the OPA4350 transition region.**
- +IN = 0.667·Vsac enters the N/P input-pair transition region, (V+)−2.4 … (V+)−1.2 V = +0.1 … +1.3 V worst case (OPA4350 DS §7.3), for positive Vsac.
- There, V_OS, CMRR and THD degrade (DS text), which shows up as a small, level-dependent residual of Vsac.
- Expect slightly worse rejection for large positive reference excursions. It is visible in data because AIN7 is recorded. No change needed.

**F-13: Charge-pump spur aliasing (bring-up check).**
- LM27762 f_SW = 1.7–2.3 MHz (DS §6.5); f_MOD = 2.56 MHz.
- Only energy within ±f_DATA/2 of k·f_MOD aliases. The fundamental is outside that, but a harmonic may coincide.
- If a fixed spur appears in the FFT with shorted inputs (J5 1–2, no flex), shift OSR or CLKIN slightly to confirm.

**F-14: DESIGN.md corrections found while verifying.** These are documentation only.
- ADC input impedance is 528 kΩ at this clock (DS eq. 4), not 330 kΩ.
- U2 pin 27 is **NC** ("leave unconnected or connect to AGND"; DS Table 5-1), not AGND. Tying it to GND is allowed.
- The input limit is now verified: AGND−1.3 V (recommended), −1.6 V (abs max).
- XTAL_DIS = 1 is recommended for power, not mandatory.

---

## 4. Summary

- **Blockers:** none in the logical circuit. The netlist implements the documented chain exactly, and the ADC absolute and full-scale limits hold even with saturated op-amps.
- **Should-fix before ordering:**
  - **F-1 (known):** 1 MHz SCLK cannot carry the fixed 10-word frame at 10 kSPS (DS §8.5.1.7 and §8.5.1.11 forbid short frames while converting). Use 5.12 MHz SCLK from the same PLL as MCO2; "~8 MHz" is not synchronous either.
  - **F-2 (known, designer-accepted):** 1 MΩ ∥ 15 pF gives a dF/dt response. Slow or static pressure is not measurable (offset ≡ 0.5 nA ≈ 20 N/s). Confirm the thesis requirement. The hedge is BOM-only (e.g. 10 MΩ + 1 nF or 1 GΩ + 220 pF on the same pads).
  - **F-3:** the signal amplitude requirement is undefined, and the chain uses about 0.2 % of FS at gain 1. Use 24/32-bit words and set the PGA per channel.
  - **F-4 (known):** no input protection; OPA4350 HBM is only ±1 kV. Ensure the touched electrode is GND and buy spare op-amps. Add 1 k series resistors in the next revision.
  - **F-5:** an H7 powered first can back-power U2 through CS/SYNC/SCLK/MOSI. Use 1 k / 330 Ω series resistors on the MCU side, and gate the outputs on DRDY going high.
