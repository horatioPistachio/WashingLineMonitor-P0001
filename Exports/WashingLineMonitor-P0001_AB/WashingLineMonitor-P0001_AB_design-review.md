# Design Review — WashingLineMonitor-P0001 Rev AB

**Date:** 2026-05-30  
**Project:** WashingLineMonitor-P0001  
**Revision:** AB ("Updated from AI review")  
**Company:** Horatio Pistachio  
**Reviewed by:** Copilot KiCad Skill (automated analysis)  
**Verdict:** ⛔ NOT READY FOR FABRICATION — 1 board-killing bug + 4 manufacturing blockers

---

## Quick-Reference: Prioritised Issues

| # | Severity | Ref | Rule | Summary |
|---|----------|-----|------|---------|
| 1 | 🔴 CRITICAL | Q1 | Pinout | PJA3401A symbol pins D/G/S ≠ actual G/S/D — LDO never receives power, board dead |
| 2 | 🔴 CRITICAL | All | SS-001 | 0/31 parts have MPNs — cannot order, cannot verify |
| 3 | 🔴 HIGH | F.Cu | FD-001 | No fiducials on front side — fine-pitch QFN (U6) cannot be reliably placed |
| 4 | 🔴 HIGH | C14/C15/C16 | PM-002 | 0.36mm from board edge — below JLCPCB 0.5mm minimum |
| 5 | 🔴 HIGH | R9 | PM-002 | 0.39mm from board edge — below JLCPCB 0.5mm minimum |
| 6 | 🟠 HIGH | U6 | TV-001 | BQ51050B (VQFN-20) has 0 thermal vias — will overheat during Qi charging |
| 7 | 🟡 MED | U1 | TV-001 | HDC2080 has only 2 thermal vias (5 minimum recommended) |
| 8 | 🟡 MED | U3 | TV-001 | ESP32-C6-MINI-1 GND pads have 0 thermal vias |
| 9 | 🟡 MED | U3/C1/C2 | PM-001 | C1, C2 inside ESP32 RF keepout — WiFi/BLE range degraded |
| 10 | 🟡 MED | — | TE-001 | 0% net test-point coverage (debug/bring-up difficult) |

---

## 1. Overview

**Design summary:** IoT washing-line monitor using an ESP32-C6-MINI-1 for WiFi/BLE connectivity, a Texas Instruments HDC2080 humidity/temperature sensor (I2C), a resistive moisture-sensing front-end (Harwin S1711-46R contacts + TLV2371 op-amp + BZX84C5V1 Zener clamp), a TI BQ51050B Qi wireless charging receiver, a 3.3V LDO (TLV75533PDBV), Li-ion battery (JST SH), and an NTC thermistor for battery temperature monitoring.

**Board:** 58.5 × 41.5mm, 2-layer, all-SMD front side (68 components) plus 2 THT back (Q3 AO6608 listed on back, SW1).  
**Analysis run:** `2026-05-30_2120` (schematic + PCB + gerbers + cross-analysis + thermal)

---

## 2. Verification Basis

| Item | Status |
|------|--------|
| `datasheets/` directory | ❌ Not present |
| MPN coverage | ❌ 0/31 unique parts (0%) |
| Datasheet-backed findings | ❌ 0 (all findings are consistency-only) |
| Q1 pinout confirmed from web search | ✅ PJA3401A Pin 1=G, Pin 2=S, Pin 3=D (allDatasheet / Octopart) |
| I2C pull-up values confirmed | ✅ from raw schematic nets |
| Cross-analysis (sch↔pcb) | ✅ 0 mismatches |

**All verification claims are internal-consistency only except where explicitly noted.** Pin-level correctness of ICs (U1, U2, U5, U6) against manufacturer datasheets has not been confirmed — the HDC2080, TLV2371, TLV75533, and BQ51050B have datasheet URLs in the schematic but datasheets were not downloaded or extracted.

---

## 3. Critical Bug — Q1 (PJA3401A) Pinout Mismatch — BOARD WILL NOT POWER UP

**Finding source:** Web-search confirmed PJA3401A datasheet (allDatasheet, Octopart, Panjit).  
**Confidence:** High — multiple independent sources agree on Pin 1=Gate, Pin 2=Source, Pin 3=Drain.

The schematic symbol description reads "P-MOSFET transistor, drain/gate/source" (i.e., Pin1=D, Pin2=G, Pin3=S). This is the convention from the KiCad standard `Q_PMOS_DGS` symbol, which Q1 was before revision AB.

**Actual PJA3401A SOT-23 pinout:**

| Physical Pin | Actual Function | Schematic Net (what is connected) | What the board actually gets |
|---|---|---|---|
| 1 | **Gate** | D (drain net → JP1 → Vbatt) | Gate driven to Vbatt |
| 2 | **Source** | G (gate net → R15 → GND) | Source pulled to GND via 10kΩ |
| 3 | **Drain** | S (source net → U5.IN) | Drain is the LDO input |

**Consequence:**
- V_GS = V_Gate − V_Source = Vbatt − 0V = +3.7V (positive)
- A P-channel MOSFET requires V_GS < V_th (≈ −0.6 V to −1.5 V) to turn ON
- Positive V_GS keeps the PMOS **permanently OFF**
- No current reaches U5 (TLV75533 LDO input)
- The 3.3V rail never comes up
- **The board is dead on first power-on**

The body diode does not rescue the circuit: with Source (actual physical pin 2) at GND and Drain (physical pin 3) at U5.IN, the PMOS body diode would require U5.IN to go negative to conduct — impossible in normal operation.

**Fix required before ordering:**
1. Replace the Q1 symbol with one whose pin-number-to-function mapping matches PJA3401A: Pin1=Gate, Pin2=Source, Pin3=Drain. Alternatively, create a corrected PJA3401A_R1 symbol.
2. Rewire the circuit so:
   - Physical Pin 2 (Source) → Vbatt (via JP1)
   - Physical Pin 1 (Gate) → R15 pull-down to GND + SW1/control signal
   - Physical Pin 3 (Drain) → U5.IN (LDO input)
3. Verify that V_GS (= V_Gate − V_Source) can be pulled sufficiently negative to turn on the PMOS during normal operation.

---

## 4. Component Summary

**70 total components, 32 unique parts**

| Type | Count |
|------|-------|
| Capacitor | 25 |
| Resistor | 18 |
| Test point | 10 |
| IC | 5 (U1 HDC2080, U2 TLV2371, U3 ESP32-C6, U5 TLV75533, U6 BQ51050B) |
| Transistor | 2 (Q1 PJA3401A PMOS, Q3 AO6608 dual FET) |
| Connector | 3 (J1/J2 Harwin S1711, J3 2×3 header) |
| Diode | 1 (D2 BZX84C5V1 5.1V Zener) |
| LED | 1 (D3) |
| Switch | 1 (SW1 A6TN-1104 slide) |
| Jumper | 1 (JP1 solder jumper, closed by default) |
| Inductor | 1 (L1, Qi coil connector WR483245-15F5-G) |
| Thermistor | 1 (TH1 NTC) |
| Battery | 1 (BT3 JST SH) |

**BOM status:** 0 MPNs set on any component. 9 components have datasheet URLs embedded in symbol properties (Q1, Q3, SW1, U1, U2, U3, U5, U6, D2). MPNs must be added before ordering.

---

## 5. Power Architecture

### 5.1 Power Tree

```
Battery (BT3) / Qi Wireless Charger (U6 BQ51050B)
  └── Vbatt rail (Li-ion, ~3.7–4.2V)
        └── JP1 (solder jumper, default closed)
              └── Q1 PJA3401A (P-channel power switch)  ← PINOUT ERROR, see §3
                    └── U5 TLV75533PDBV (3.3V LDO)
                          └── 3V3 rail → U3 ESP32-C6-MINI-1

Sensor power gating (ESP32 GPIO-controlled):
  U3.IO1 → HDC_POWER → U1 HDC2080 VDD, R13/R14 I2C pull-ups, C24 decoupling
  U3.IO5 → OP_AMP_POWER → U2 TLV2371 V+, C23 decoupling
  U3.IO3 → SENSOR_PWR → R1 (moisture sense excitation)
```

### 5.2 Power Budget

| Rail | Estimated Load | Regulator |
|------|---------------|-----------|
| 3V3 | 240mA (ESP32 peak WiFi TX) | TLV75533PDBV LDO |
| Vbatt | charging + load | BQ51050B (Qi input) + BT3 |
| HDC_POWER | ~710µA (HDC2080 50µA + I2C pull-ups 660µA) | ESP32 GPIO IO1 |
| OP_AMP_POWER | ~50µA TLV2371 Iq | ESP32 GPIO IO5 |

GPIO-powered sensors: The ESP32-C6 GPIO can source up to 40mA; the HDC_POWER and OP_AMP_POWER loads (~760µA total) are well within limits. This is a valid low-power design pattern.

### 5.3 Analyzer PP-001 Findings — False Positives

The analyzer raised PP-001 ("no DC path to power rail") for U1.VDD, U2.V+, and U5.IN. These are false positives:
- **U1.VDD and U2.V+** are intentionally GPIO-powered (HDC_POWER and OP_AMP_POWER nets). The analyzer does not model GPIO outputs as power sources.
- **U5.IN** is powered through Q1 (MOSFET). The analyzer does not model MOSFET conduction paths. The physical path is Vbatt → JP1 → Q1 → U5.IN — this is correct in design intent but incorrect in implementation due to the Q1 pinout error (§3).

### 5.4 U5 Enable Circuit

SW1 (A6TN-1104 slide switch) and R16 (500kΩ to GND) control the U5 enable pin:
- **Switch open:** R16 pulls U5.EN to GND → LDO disabled
- **Switch closed:** SW1 connects U5.IN (LDO input rail) to U5.EN → LDO enabled

This is a valid design. The PU-001 warning ("U5 EN missing pull-up") is a **false positive** — R16 is a pull-down, which is correct for this enable-when-switch-closed topology. No fix needed.

---

## 6. Schematic Analysis

### 6.1 Moisture Sensing Circuit

**Signal path:** Harwin J1 → Vsens net → D2 (BZX84C5V1 Zener, 5.1V clamp) → U2.+ (op-amp non-inverting) → U2 TLV2371 → SENSOR_OUTPUT → RC filter (R6 100Ω / C19 100nF, fc=15.9kHz) → U3.IO4 (ESP32 ADC).

Detected RC filters:
- R6/C19: fc = 15.9kHz (anti-aliasing for moisture sensing output)
- R13/C24: fc = 159Hz (HDC_POWER decoupling network — this appears as an RC filter but is the sensor power supply decoupling)
- R14/C24: fc = 159Hz (same C24 shared with R13)

The Zener clamp (D2, 5.1V) protects U2's non-inverting input from overvoltage when conductive material contacts J1. The op-amp output feedback is through R6 (the 100Ω resistor in a virtual-ground configuration). The SENSOR_PWR signal (U3.IO3) provides excitation through R1 (500kΩ) to the Vsens node.

**Potential concern:** R1 is 500kΩ from SENSOR_PWR to Vsens. This is a very high excitation resistance, which limits the moisture sensing current to <10µA at 3.3V. This may be intentional for very resistive/capacitive moisture sensing, but verify the expected sensor resistance range against this value.

### 6.2 I2C Bus

| Net | Pull-up | Value | To Rail |
|-----|---------|-------|---------|
| SDA (U3.IO6 ↔ U1.SDA) | R13 | 10kΩ | HDC_POWER |
| SCL (U3.IO7 ↔ U1.SCL) | R14 | 10kΩ | HDC_POWER |

I2C pull-ups are tied to HDC_POWER (GPIO-controlled sensor supply). When the sensor is powered off, the I2C bus floats — this prevents phantom current through the bus when the sensor is off. The ESP32 internal pull-ups should not be enabled simultaneously.

### 6.3 Battery Measurement Circuit (Q3 / AO6608)

The AO6608 dual NMOS+PMOS is used for battery voltage measurement. Key connections:
- Q3.NMOS.G (Pin1) and Q3.PMOS.S (Pin2) share the same net (__unnamed_22) with R10 (500kΩ to Vbatt)
- Q3.PMOS.G (Pin3) → MEAS_BATT (U3.IO8)
- Q3.NMOS.D (Pin6) → Vbatt

The PMOS gate (MEAS_BATT) is controlled by the ESP32 to switch measurement on/off. R18 (10kΩ) and R19 (10kΩ) form a 1:1 voltage divider from Vbatt to Battery_Voltage, which connects to U3.IO2 (ESP32 ADC). C22 (100nF) decouples the Battery_Voltage node.

**Note:** The NMOS gate tied to PMOS source (both on __unnamed_22) is an unusual configuration. The design intent should be confirmed: this appears to use the PMOS to control power to the resistor divider, with the NMOS potentially acting as a diode or not used. No errors were detected in the net connections, but full verification of the AO6608 application circuit requires the schematic diagram view.

### 6.4 Qi Wireless Charging (U6 BQ51050B)

The BQ51050B is a 5W single-mode Qi wireless power receiver. Key connections verified:
- AC1/AC2 → Qi coil (L1 via JST SH connector) with resonance capacitors (C3/C4/C5/C6, C7 on AC2 side; C8/C9/C10/C12/C13 on AC1/coil side)
- RECT (Pin18) → __unnamed_37 (C14, C15, C16, R9) — rectifier output filter capacitors
- BAT (Pin4) → Vbatt — battery charge output
- TS/CTRL (Pin13) → NTC_BATT → TH1 (NTC thermistor) + NTC1 test point — temperature protection
- CHG (Pin7, active-low open-drain) → CHARGE_STATUS → U3.IO15 — charge status to ESP32
- EN2 (Pin11) → QI_EN2 → U3.IO14 — wireless power enable from ESP32
- AD (Pin9) → GND, FOD (Pin14) → GND — analog detection grounded per typical application
- TERM (Pin10) → R8 (2.4kΩ) — termination resistor setting
- ILIM (Pin12) → R7 (430Ω) — current limit setting

**Critical concern — RECT filter capacitors at board edge:** C14, C15, C16 (each 10µF, on the RECT output) and R9 (20kΩ on RECT pin) are 0.36mm from the board edge (see §7, PM-002). These must be moved inward by at least 0.2mm.

### 6.5 ESP32-C6-MINI-1 GPIO Assignment

| GPIO | Net | Function |
|------|-----|---------|
| IO1 | HDC_POWER | HDC2080 VDD + I2C pull-up rail |
| IO2 | Battery_Voltage | Battery voltage ADC |
| IO3 | SENSOR_PWR | Moisture sensor excitation enable |
| IO4 | SENSOR_OUTPUT | Moisture sensor ADC input |
| IO5 | OP_AMP_POWER | TLV2371 VDD |
| IO6 | SDA | I2C data |
| IO7 | SCL | I2C clock |
| IO8 | MEAS_BATT | AO6608 battery measurement switch |
| IO13 | HDC_POWER (also) | HDC2080 VDD (IO1 and IO13 both drive this net) |
| IO14 | QI_EN2 | Qi charging enable |
| IO15 | CHARGE_STATUS | Qi charge status input |
| IO18 | R12 → D3 | LED driver |
| IO0 | ESP_BOOT | Boot mode strap |
| EN | ESP_EN | Module enable (R2 10kΩ pull-up to 3V3, J3 override) |

**Note:** Both IO1 and IO13 appear on the HDC_POWER net. Having two GPIOs drive the same rail requires careful firmware management (one should be output, the other input or both output to same state). Verify this is intentional — it could indicate a net naming collision in the schematic.

---

## 7. PCB Layout Analysis

### 7.1 Board Summary

| Parameter | Value |
|-----------|-------|
| Board size | 58.5 × 41.5mm |
| Layers | 2 (F.Cu + B.Cu) |
| Total footprints | 70 (68 front, 2 back) |
| SMD count | 68 |
| THT count | 1 (SW1 slide switch) |
| Via count | 72 |
| Routing | Complete (0 unrouted nets) |
| DFM tier | Standard (JLCPCB standard rules met) |
| Track segments | 324 |

### 7.2 Component Edge Clearance — ERRORS

| Ref | Distance to edge | Limit | Finding |
|-----|-----------------|-------|---------|
| C14 | 0.36mm | 0.50mm min | ❌ PM-002 |
| C15 | 0.36mm | 0.50mm min | ❌ PM-002 |
| C16 | 0.36mm | 0.50mm min | ❌ PM-002 |
| R9 | 0.39mm | 0.50mm min | ❌ PM-002 |
| J1 | 0.55mm | 0.50mm min | ⚠️ PM-002 (margin only 0.05mm) |
| J2 | 0.55mm | 0.50mm min | ⚠️ PM-002 (margin only 0.05mm) |

C14/C15/C16 are BQ51050B RECT filter capacitors. R9 is on the same RECT net. **All four must be moved ≥0.15mm toward the board interior before sending to fab.** J1/J2 (Harwin S1711-46R moisture probes) are marginally within limits but should be reviewed in context of board routing.

### 7.3 Fiducials — ERROR

No fiducials on F.Cu. The front side has 67 SMD components including the BQ51050B (VQFN-20, 0.5mm pitch) and HDC2080 (S-PWSON-N6, fine pitch). JLCPCB and all other PCB assembly houses require ≥2 (ideally 3) fiducials on each assembly side for optical alignment. Without fiducials, pick-and-place machines cannot compensate for board stretch or rotation, making fine-pitch placement unreliable.

**Add 3 fiducials to F.Cu** (e.g., 1mm copper circles with 3mm no-copper keepout, placed at three corners or asymmetrically to enforce orientation).

### 7.4 Thermal Vias — HIGH SEVERITY

| Ref | IC | Vias Found | Recommended Min | Finding |
|-----|----|-----------|-----------------|---------|
| U6 | BQ51050B (VQFN-20) | **0** | 5 | ❌ TV-001 |
| U1 | HDC2080 | 2 | 5 | ⚠️ TV-001 |
| U3 | ESP32-C6-MINI-1 | 0 (multiple pads) | 5 | ⚠️ TV-001 |

**U6 (BQ51050B) is the most critical.** The BQ51050B is a switching wireless charging receiver that handles all AC/DC conversion. At 5W input with ~85% efficiency, it dissipates ~0.75W into the VQFN-20 exposed pad. Without thermal vias, the pad cannot transfer heat to the ground plane. TI's BQ51050B application note recommends a via array under the exposed pad. **Add minimum 4–9 thermal vias (0.3mm drill, 0.5mm annular ring) under the exposed pad before ordering.**

U3 and U1 should also have thermal vias added under their ground/exposed pads.

### 7.5 RF Keepout Violations (U3 ESP32-C6-MINI-1)

The ESP32-C6-MINI-1 module courtyard includes an antenna keepout zone. The analyzer detected courtyard overlaps with:

| Component | Overlap Area |
|-----------|-------------|
| C1 (22µF 0805) | 7.37mm² |
| C2 (22µF 0805) | 7.37mm² |
| SW1 (slide switch, THT) | 28.9mm² |
| SenseOut1 test point | trace |
| 3V3 power symbol | trace |

C1 and C2 inside the antenna keepout will detune the ESP32 antenna and reduce WiFi/BLE range. Espressif's antenna keepout requires no copper or components within the module's antenna area. **Move C1, C2 outside the antenna keepout area.** SW1's large overlap should also be checked against the keepout boundary.

### 7.6 Copper Presence (Expected Voids)

The analyzer confirmed no opposite-layer copper under:
- SW1 (expected — THT switch, no GND pour conflict needed)
- TH1, T2, Vsense1 (expected — analog/sensing components isolated from GND pour noise)

### 7.7 Test Point Coverage

10 test point footprints exist in the design, covering specific nets (SenseOut1, Vsense1, T2, GND, BattMeas, Vbatt ×2, 3V3, NTC, AD-EN). The analyzer reports 0/60 net coverage. The existing test points are useful for signal-chain debugging (moisture sensing input/output, battery voltage, Qi AD-EN) but **SDA, SCL, SENSOR_OUTPUT, CHARGE_STATUS, ESP_EN, and all power rails should have accessible test points or solder pads for bring-up**.

### 7.8 Passive Orientation

13 passives on F.Cu deviate from the 90° majority orientation. This does not affect function but increases assembly visual inspection difficulty. No action required, but worth addressing in a future revision.

---

## 8. Gerber Analysis

**Result: PASS — no findings**

| Layer | File | Status |
|-------|------|--------|
| F.Cu | -F_Cu.gtl | ✅ |
| B.Cu | -B_Cu.gbl | ✅ |
| F.Mask | -F_Mask.gts | ✅ |
| B.Mask | -B_Mask.gbs | ✅ |
| F.Paste | -F_Paste.gtp | ✅ |
| B.Paste | -B_Paste.gbp | ✅ |
| F.SilkS | -F_Silkscreen.gto | ✅ |
| B.SilkS | -B_Silkscreen.gbo | ✅ |
| Edge.Cuts | -Edge_Cuts.gm1 | ✅ |
| PTH drill | -PTH.drl | ✅ |
| NPTH drill | -NPTH.drl | ✅ |

Board dimensions confirmed from gerbers: 58.55 × 41.55mm (matches PCB file: 58.5 × 41.5mm ✅). Layer set complete, no missing layers, no alignment issues.

---

## 9. Cross-Domain Analysis

**Result: 0 findings** — schematic and PCB net assignments are internally consistent. All nets that appear in the schematic are present in the PCB. No missing connections detected.

---

## 10. Thermal Analysis

**Result: 0 findings (score 100/100)**

At the estimated power levels (ESP32 WiFi TX ~240mA × 3.3V = ~0.8W; sensor loads <1mA), no thermal violations were flagged at 25°C ambient. The thermal analyzer's estimates are conservative and do not account for U6 (BQ51050B) wireless charging losses (see §7.4 — U6 has no thermal vias, this is a layout-level concern not captured by the thermal estimator).

---

## 11. Simulation Verification

**Not performed** — no SPICE simulator (ngspice/LTspice/Xyce) found in environment. Circuit-level simulation of the moisture sensing op-amp configuration, RC filter cutoffs, and Qi resonance network could not be verified.

---

## 12. EMC Pre-Compliance

**Not performed** — EMC analysis script not installed. Given the ESP32-C6 WiFi/BLE module and Qi wireless power (6.78MHz resonant switching), EMC pre-compliance review is recommended before CE/FCC testing.

---

## 13. Lifecycle Audit

**Not performed** — no MPNs set on any component. Add MPNs and re-run lifecycle audit before production quantities.

---

## 14. Previous Review Delta (Rev AA → Rev AB)

Changes detected between analysis run `2026-05-29_1352` (AA-era) and `2026-05-30_2120` (AB):

| Change | Detail |
|--------|--------|
| Added | J2 (S1711-46R Harwin connector — second moisture probe) |
| Added | R20 (500kΩ) |
| Modified | J1: value changed from generic `Conn_01x02_Socket` → `S1711-46R`; footprint updated |
| Modified | Q1: value changed from `Q_PMOS_DGS` (generic) → `PJA3401A_R1_00001` (specific part); footprint added — **this change introduced or revealed the pinout mismatch** |
| Modified | Q3: footprint `Package_SO:TSOP-6` added |
| Modified | SW1: value from `SW_DIP_x01` → `A6TN-1104`; footprint added |
| Modified | L1, BT3: footprints updated from solder-wire to JST SH connector |
| Net stats | +2 components, −2 nets, +9 wires, −1 no-connect |
| Resolved | Single-pin net `~{MEAS_BATT}` on Q3.G resolved (net renamed) |

The assignment of the PJA3401A to the Q_PMOS_DGS-style symbol is the source of the critical pinout error (§3).

---

## 15. False Positive Summary

| Finding | Rule | Assessment |
|---------|------|-----------|
| U1.VDD no DC path | PP-001 | ✅ False positive — intentional GPIO power gating |
| U2.V+ no DC path | PP-001 | ✅ False positive — intentional GPIO power gating |
| U5.IN no DC path | PP-001 | ✅ False positive — MOSFET path not modeled by analyzer |
| U5.EN missing pull-up | PU-001 | ✅ False positive — R16 provides correct pull-down; topology is valid |
| HDC_POWER no declared source | RS-001 | ✅ False positive — GPIO-driven, intentional |
| OP_AMP_POWER no declared source | RS-001 | ✅ False positive — GPIO-driven, intentional |
| U3 courtyard overlaps (3V3, SenseOut1) | PM-001 | ✅ False positive — power symbol / test point, not physical body |

---

## 16. Positive Findings

- ✅ Complete gerber set, no layer gaps, drill files present
- ✅ Routing 100% complete (0 unrouted nets)
- ✅ Cross-domain (sch↔pcb) 0 findings — design is internally consistent
- ✅ DFM: Standard tier, 0 violations for standard JLCPCB rules
- ✅ NTC thermistor correctly wired to BQ51050B TS/CTRL for battery overtemperature protection
- ✅ Battery voltage monitoring via R18/R19 resistive divider → ESP32 ADC (IO2)
- ✅ Qi charge status (CHG) and enable (EN2) correctly connected to ESP32 GPIOs
- ✅ I2C bus has 10kΩ pull-ups on both SDA and SCL, tied to switchable sensor supply
- ✅ Zener clamp (D2, 5.1V) protects op-amp input from moisture probe overvoltage
- ✅ GPIO power gating for sensors reduces standby current — good IoT design practice
- ✅ CERT-001: Wireless module detected (ESP32-C6) — device will require FCC/CE/ACMA certification

---

## 17. Required Actions Before Fabrication

### Blockers (must fix before ordering):

1. **[ ] Fix Q1 pinout** — Rewire PJA3401A so Source→Vbatt, Gate→R15/SW1-control, Drain→U5.IN. Update or replace symbol. Verify against PJA3401A datasheet Pin 1=G, Pin 2=S, Pin 3=D.

2. **[ ] Add MPNs to all 31 unique parts** — Required for ordering, BOM validation, and lifecycle audit.

3. **[ ] Add fiducials to F.Cu** — Minimum 2, recommended 3 (1mm copper circle, 3mm keepout). Required for SMD assembly of BQ51050B VQFN-20.

4. **[ ] Move C14, C15, C16, R9 inward by ≥0.15mm** — Currently 0.36–0.39mm from edge; fab minimum is 0.5mm.

### High priority (fix before assembly):

5. **[ ] Add thermal via array under U6 (BQ51050B) exposed pad** — Minimum 4–9 vias, 0.3mm drill, 0.5mm pad. Critical for reliability of Qi charging.

6. **[ ] Move C1, C2 outside ESP32 antenna keepout** — 22µF bulk caps inside keepout will degrade WiFi/BLE range.

### Recommended (fix before production):

7. **[ ] Add thermal vias under U1 (HDC2080)** — Currently 2, recommend 5.

8. **[ ] Add thermal vias under U3 (ESP32-C6-MINI-1) GND pads**.

9. **[ ] Verify IO1 vs IO13 both driving HDC_POWER** — Confirm this is intentional in firmware.

10. **[ ] Add MPNs, then run lifecycle audit and SPICE simulation**.

11. **[ ] Download datasheets and verify U6 application circuit** — The BQ51050B RECT capacitor values, BOOT1/BOOT2 capacitor values, TERM resistor, and ILIM resistor should be verified against TI's recommended values in the BQ51050B datasheet.

---

## 18. Analyzer Coverage

| Analyzer | Run | Status |
|----------|-----|--------|
| `analyze_schematic.py` | 2026-05-30_2120 | ✅ Complete |
| `analyze_pcb.py --full` | 2026-05-30_2120 | ✅ Complete |
| `analyze_gerbers.py` | 2026-05-30_2120 | ✅ Complete |
| `cross_analysis.py` | 2026-05-30_2120 | ✅ Complete |
| `analyze_thermal.py` | 2026-05-30_2120 | ✅ Complete |
| `analyze_emc.py` | — | ❌ Skill not installed |
| SPICE simulation | — | ❌ No simulator found |
| Lifecycle audit | — | ❌ No MPNs (prerequisite) |
| Datasheet verification | — | ❌ No `datasheets/` directory, no MPNs |
| Diff vs prior run | 2026-05-29_1352 → 2026-05-30_2120 | ✅ Complete |

*All findings are internal-consistency only. Manufacturer-verified conclusions require downloading datasheets after adding MPNs.*
