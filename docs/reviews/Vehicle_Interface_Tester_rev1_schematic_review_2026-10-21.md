# Vehicle Interface Tester rev1 schematic review — 2026-10-21

**Scope: schematics only.** No PcbDoc has been laid out against these sheets (the 2.9 MB
PcbDoc on disk is from 10-19 and predates every sheet), so §9 (fabrication) and the
PcbDoc↔netlist checks are deferred to the pre-order pass.

Inputs: schematic PDF and Protel netlist exported **2026-10-21 14:54–14:58** from the working
tree (10 of 13 `.SchDoc` files modified since commit `2264ae9`, not committed); BOM `.xlsx`
14:58. All thirteen sheets were parsed directly from the `.SchDoc` records as well as viewed
rendered, and every connection below was taken from the netlist, not from the drawing.

Datasheets present (`UpRev_interface_PCB/Datasheets/`): STM32WB1MMCH6 module (DS14096),
TJA1027, TDK MCZ1210AH, SQ2309ES, BZT52C20, 1SMA4736A, MMBT3904, SMAJ26CA.
**Missing:** TPS5450, TPS54302, PMEG3050EP, ESD1LIN24, SESD0402, the H753 dev board that
plugs into J1, and the Molex/Samtec/GCT connector drawings. Where a missing datasheet forced
an estimate the verdict says so.

Firmware reviewed: none exists yet. §7 is therefore a list of constraints for the firmware,
not a check of it.

Board under test (for §8 cross-checks): `UpRev_interface_PCB` netlist of 2026-10-13.

## Headline

Seven FAILs. Three of them mean the board does nothing when first powered, so they are
worth fixing before any layout starts:

| | |
|---|---|
| **F1** | 14 V buck feedback divider is installed upside-down: **Vout = 1.34 V**, not 14 V. |
| **F2** | Input over-voltage cut-off trips at **≈21 V**. On a 24 V supply Q1 is held off and nothing downstream powers. At 20 V it is marginal. |
| **F3** | Five power nets have no source. The 5 V buck output goes nowhere; `+5V`, `VCC`, `3V3_D`, `24V_CLAMPED` are floating; `VBUS` exists only if the dev board sources it. |
| **F4** | USB-C port has sink pull-downs (Rd) but the board is the host. Needs Rp. |
| **F5** | BLE UART lands on H753 pads **PA0_C / PA1_C**, which are ADC-only and have no UART function. |
| **F6** | BOOT0 on the BLE module sits at a 1K/1K divider (1.65 V). The sheet says R10 is NF; the BOM still orders it. |
| **F7** | Rig SWD header J11 has no VTref: ST-Link will refuse to connect. |

Everything else that could be checked numerically passed; see the table. The big open item
is the dev board: without its schematic, which rails it supplies versus consumes (F3) is a
guess.

---

## Findings

| # | Section | Item | Verdict | Number / reasoning | Fix |
|---|---------|------|---------|--------------------|-----|
| **F1** | 2, 6 | **14 V buck divider orientation (U3 TPS5450)** | **FAIL** | Netlist: `14V` net holds R29-2 (**2K**); `GND` holds R30-1 (**21K**); VSENSE = R29-1/R30-2. So the 2K is on top and the 21K on the bottom: Vout = 1.221 × (1 + 2/21) = **1.34 V**. The sheet note "1.221 × (1 + 21K/2K) = 14.04 V" describes the intended orientation. The orienter original (`Orienter.NET`) has it right: R39 22K on the 16V net, R40 1.8K to GND. | Swap the two values: R29 = 21K, R30 = 2K. Re-check the note. |
| **F2** | 1, 2 | **Input OV cut-off threshold vs the 20–24 V supply** | **FAIL** | The "20V_CLAMPED" stage is not a clamp; it is an input-sensed cut-off. D3 (BZT52C20) feeds Q3's base from the **raw input** (NetC1_1), Q3 drives Q2 (PNP), Q2 shorts Q1's gate to its source. Q2 starts conducting when Q3 drops ≈0.6 V across R3 (60 µA), which needs Ve ≈ 0.6 V, so **Vth ≈ Vz + 0.65 + 0.6 ≈ 21.3 V** nominal. BZT52C20 Vz is 18.8–21.2 V at 5 mA and runs here at ≈125 µA (R4), i.e. 0.5–1 V lower, so the real band is roughly **19–22 V**. At Vin = 24 V: Ve ≈ (24 − 20 − 0.65)/2 = 1.7 V, Ic(Q3) ≈ 170 µA, 1.7 V across R3, Q2 saturated, **Q1 off, board dead**. At 20 V with a low-side zener (18.5 V): Ic ≈ 43 µA, Q2 partly on, Q1 linear and dissipating. There is no hysteresis, so near threshold it chatters. | Decide the supply voltage, then set the threshold ≥2 V above it and below the TPS54302's 28 V max. Recommended: replace D3+Q3 with a **TL431** (cathode to Q2's base through R3, ref from a 95.3K/10K divider off the input) → **26.3 V ± 0.3 V**. A BZT52C24 keeps the topology but lands at 24–27 V, which can trip on a 24 V supply. Keep R6/C1 (10 ms soft gate) as is. |
| **F3** | 8 | **Power nets with no source** | **FAIL** | From the netlist: `5V` = {C15, C16, C17, C18, L2, R23, R25} only — the buck powers its own LED. `+5V` = {J1-1, J1-3} only. `VCC` = {J1-2, J1-4} only. `3V3_D` = {C4, C5, J8-1, MD1-16/18/20, R10, R14, R15, R16} — no regulator. `24V_CLAMPED` = {J4-1, J4-2, J4-3} only — "POW TO DEV BOARD" is fed by nothing; the input stage produces `20V_CLAMPED`. `VBUS` = {C3, D7, J3-1, J7} — sourced only if the dev board drives J3-1. J4-4 "ENABLE P.S." is unconnected. | Wire by intent: (a) `24V_CLAMPED` → `20V_CLAMPED` (or delete J4 if the dev board is fed 5 V instead); (b) `+5V` ← `5V` if the buck feeds the dev board, else delete the buck; (c) `3V3_D` ← dev-board 3.3 V (`VCC`) or a local LDO off 5 V — the module plus pull-ups is <50 mA; (d) `VBUS` ← 5 V through a TPS2553 (see N2) or confirm the dev board sources it. Then rename so each rail has one name. |
| **F4** | 8 | **USB-C CC configuration for a host** | **FAIL** | R7, R8 = 5.1K from CC1/CC2 to GND (netlist `NetJ7_A5`, `NetJ7_B5`). 5.1K to ground is **Rd**, the sink advertisement. A host that sources VBUS must present **Rp** on each CC: 56K to 5 V (default USB power) or 22K (1.5 A). The product (`UpRev_interface_PCB` J1) presents Rd; two Rd ends never negotiate a source. The product will still enumerate if VBUS is forced, but it is out of spec and any CC-aware device will refuse. | R7, R8 → 56K to `VBUS`. |
| **F5** | 5, 7 | **BLE UART on H753 `_C` pads** | **FAIL (downgraded 2026-10-21 pm: workable, not recommended)** | `BLE_TX` → J1-20, `BLE_RX` → J1-22; the dev-board sheet labels those pins **PA0_C** and **PA1_C**. On STM32H743/753 the `_C` pads are the direct ADC inputs; per the datasheet footnote they connect to PA0/PA1 **through an analog switch** (SYSCFG_PMCR PA0SO/PA1SO), and the PA0/PA1 alternate functions are available on the `_C` pad only while that switch is closed. So UART4 (PA0 TX / PA1 RX, AF8) can reach these pads, but through a switch with series resistance, at reduced edge rate, and only if firmware leaves the switch closed (reset default per RM0433; confirm). It also forfeits PA0/PA1 as the ADC pads. The module side is fine: PB5 = LPUART1_TX and PA3 = LPUART1_RX on STM32WB15 (AF8, from memory — see Not checked). | Preferred: move to a real UART pair on J1, e.g. **PA9/PA10** (pins 25/27, USART1 TX/RX, AF7), both currently unused. If the pins must stay, document the SYSCFG switch requirement in the firmware notes and keep the baud ≤ 115200. Confirm against the dev board's own schematic that header pins 25/27 really are PA9/PA10. |
| **F6** | 5, 6 | **BOOT0 divider and BOM mismatch** | **FAIL** | `NetMD1_22` = MD1-22, R10-1 (1K to 3V3_D), R11-1 (1K to GND): BOOT0 = **1.65 V = VDD/2**, exactly at the sampling threshold, plus 1.65 mA wasted. The sheet marks R10 "NF", but the BOM row "R10, R11, R19 | 1K | qty 3" orders it. | Set R10's DNP/fitted parameter (not just the text) so it drops out of the BOM; R11 alone pulls BOOT0 low. 10K is enough. |
| **F7** | 8 | **Rig SWD header VTref** | **FAIL** | J11-1 (VCC) and J5-1 ("3V3 IN") both carry the no-connect cross. ST-Link V2/V3 measures target voltage on pin 1 and will not connect ("target voltage too low") without it. | Wire J11-1 to J5-1 so the dev board's 3V3 reaches the header. J10-1 (product programmer) can stay NC since the dev board bit-bangs it, but routing the product's 3V3 there would give a "DUT powered" sense input — optional. |
| **N1** | 3, 4 | **14 V output ripple / no ceramic** | **minor** | `14V` net has only C26 (220 µF electrolytic). ΔIL = 14 × (1 − 14/24)/(15 µH × 500 kHz) = **0.78 A** at 24 V in, 0.56 A at 20 V. With a general-purpose electrolytic ESR of 0.2–0.4 Ω that is **150–300 mV p-p** on the "vehicle battery". The orienter sheet is the same, so this is inherited, not new. | Add a 10 µF/50 V 1206 ceramic across C26 (ripple → ≈20 mV). Keep the electrolytic — the TPS5450 needs its ESR zero (≈2.4 kHz, next to the 2.8 kHz LC pole, which is the TI-intended arrangement). |
| **N2** | 8 | **VBUS has no switch, limit or soft-start; HOST_DET unconnected** | **minor / design decision** | `VBUS` feeds J7 straight from J3-1 with only D7 (6.8 V zener) and C3. `USB_HOST_DET` (J3-4) is on the top-level sheet symbol but on no net. The earlier recommendation (TPS2553, EN + FAULT to the dev board, ILIM ≈47K for 500 mA) is not in the schematic yet. | Add the switch once F3(d) decides where 5 V comes from. Decide what the dev board expects on HOST DETECT. |
| **N3** | 6 | **Molex PicoBlade Comment ↔ MPN swap** | **minor** | J3 (5-pos): MPN param 532610571 (correct, 5 ckt) but Comment/BOM "Name" = 532610671 (6 ckt). J4, J5 (6-pos): MPN 532610671 (correct) but Comment = 532610571. Netlist footprints are right (MOLEX_532610571 / SD_06). Only the LCSC column drives a JLC order, so this is cosmetic unless someone orders from the Comment. | Fix the Comment fields; verify LCSC C17256390 = 53261-0571 and C17393620 = 53261-0671 (not checked, offline). |
| **N4** | 6 | **C26 package text** | **minor** | BOM Package column says "SMD,D6.3xL7.7mm" for a part described and footprinted as D10×10.2 mm (`Cap D10xL10.2mm` in the netlist). Inherited from the orienter library entry. | Fix the library parameter; harmless to the build. |
| **N5** | 7 | **Product SWD lines driven while the product is unpowered** | **firmware constraint** | `PRODUCT_SW_CLK/DIO/RST/SWO` run from dev-board GPIO (PD11/PD12/PD13/PA3) to J10 with no series resistance. When `14V_ENABLE` is low the product is off; any high level on these lines back-powers its 3V3 rail through the H573's ESD diodes and invalidates the power-off test. | Firmware: tri-state (analog/input, no pull) all four before dropping 14 V. Optionally 100 Ω series on each. Same rule for BLE_RESET into the module only if 3V3_D is ever switched. |
| **N6** | 8 | **K-line role** | **PASS, noted** | Tester runs the TJA1027 as a slave (R12/R13/D8 DNP — correctly absent from the BOM). The product's K-line (`ISO-K_LINE` R34/R35 + D on its +VBAT) is the 1 kΩ master pull-up, so the bus has one strong pull-up as ISO 9141 expects. | None. If an ECU with a 510 Ω pull-up is ever emulated, fit R12/R13 and change to 1K/1K → 510 Ω equivalent. |
| **N7** | 4 | **LED currents** | **PASS** | D12: (14 − 2)/13K = 0.9 mA. D11: (5 − 2)/2.4K = 1.25 mA. Visible, dim. | — |

### Section 1 — rails and absolute maximums (sweep)

| Part | Rail / pin | Limit | Design value | Verdict |
|---|---|---|---|---|
| U2 TPS54302 | VIN | 4.5–28 V op (30 V abs) | 20–24 V (`20V_CLAMPED`), TVS clamps at 42 V only if F2's cut-off has already removed the input | PASS at 24 V; 4 V to op-max. Datasheet not on disk — limits from memory |
| U2 | EN | ≤ 7 V (memory) | 24 × 22/322 = 1.64 V | PASS |
| U2 | UVLO from R24/R26 | — | Vin(start) ≈ 1.2 × 322/22 ≈ **17.6 V** (assumes 1.2 V EN threshold, ignores internal pull-up current) | PASS (estimate); 2.4 V under a 20 V rail |
| U3 TPS5450 | VIN | 5.5–36 V | 20–24 V | PASS |
| U3 | ENA | internal pull-up, open = enabled | U3-5 on no net; identical on the orienter (U10-5), which runs on hardware | PASS by precedent |
| U3 | VSENSE | — | 1.34 V (F1) | FAIL (F1) |
| Q1, Q4 SQ2309ES | VDS | −60 V | 24 V / 14 V | PASS |
| Q1 | VGS | ±20 V | clamped to −12 V by D2 | PASS |
| Q4 | VGS | ±20 V | −14 V (gate to GND through Q5 + 1K) | PASS |
| Q2 MMBT3906, Q3/Q5 MMBT3904 | VCEO | 40 V | Q2 ≤ 12 V (zener-clamped), Q3 ≤ 24 V, Q5 ≤ 14 V | PASS |
| D1, D9 PMEG3050EP | VR | 30 V | 24 V reverse-polarity case (D1); 14 V (D9) | PASS (datasheet not on disk) |
| D3 BZT52C20 | P | 500 mW | ≈125 µA → 2.5 mW | PASS |
| D4 SMAJ26CA | standoff | 26 V | 24 V max input | PASS |
| D7 1SMA4736A | leakage at 5 V | IR spec at VR ≈ 5.2 V, µA | VBUS 5.0–5.25 V | PASS |
| D10 ESD1LIN24 | standoff | 24 V | K-line swings 0–13.6 V | PASS |
| D5/D6 SESD0402 7 V | standoff | ≥ 5 V | 3.3 V signalling | PASS |
| U1 TJA1027 | VBAT | 5–27 V | 14 − 0.4 (D9) = 13.6 V | PASS |
| U1 | TXD/SLP_N/RXD with VBAT = 0 (14 V switched off) | abs max −0.3…+7 V, not VBAT-referenced | 3.3 V pull-ups applied | PASS |
| U1 | TXD VIL/VIH | 0.8 / 2.0 V | H7 3.3 V CMOS | PASS |
| MD1 STM32WB1MMC | VDDA, VBAT, VDDSMPS | 1.71–3.6 V | 3V3_D (unsourced, F3) | PASS once F3 is fixed |
| MD1 | BOOT0 | logic input | 1.65 V | FAIL (F6) |
| R18 2.2K 1206 | P | 250 mW | 14²/2200 = 89 mW | PASS |
| R1 120R 1206 | P | 250 mW | CAN dominant ≈ 2 V²/120 = 33 mW | PASS |
| D1 | dissipation | SOD-128 ≈ 1 W practical | 0.45 V × ≈0.5 A total input current ≈ 0.23 W | PASS |

### Section 2 — scaling

No analog chains on this board. Dividers:

| Divider | Transfer | Result | Verdict |
|---|---|---|---|
| 14 V (R29/R30) | 1.221 × (1 + Rtop/Rbot) | 1.34 V as built; 14.04 V with the values swapped | FAIL (F1) |
| 5 V (R25 200K / R27 27K, C16 47 pF feed-forward) | 0.596 × (1 + 200/27) | 5.01 V | PASS |
| Input cut-off (D3/Q3/Q2) | Vz + 2 × Vbe | ≈ 21.3 V nominal, 19–22 V band | FAIL (F2) |
| 5 V UVLO (R24/R26) | VEN × 322/22 | ≈ 17.6 V | PASS (estimate) |

### Section 3 — loops

Both regulators are internally compensated parts used at the TI reference values, and both
sheets are copies of boards running on the bench (orienter 5 V and stepper rails). Numeric
loop models need the datasheet compensation internals, which are not on disk:

| Loop | Verdict | Reasoning |
|---|---|---|
| TPS54302 5 V: L2 10 µH, 44 µF ceramic + 220 µF, C16 47 pF feed-forward | PASS by precedent | identical to the orienter 5 V sheet. ΔIL = 5 × (1 − 5/24)/(10 µ × 400 k) = 0.99 A; L2 rated 5.5 A |
| TPS5450 14 V: L3 15 µH, 220 µF electrolytic | PASS by precedent, see N1 | LC pole 1/(2π√(15µ·220µ)) = 2.8 kHz, ESR zero ≈ 2.4 kHz at 0.3 Ω — the arrangement the TPS5450 internal compensation expects. ΔIL 0.78 A; L3 rated 7 A |
| Q1 OV stage | no loop | sensed from the input, so it is a comparator without hysteresis, not a regulator (F2) |

### Section 5 — package and pinout

| Item | Verdict | Evidence |
|---|---|---|
| Q1 orientation as a high-side switch | PASS | source (pin 2) on input, drain (pin 3) to D1. Body diode points input→load only when reversed, so the channel blocks when Q2 turns it off; reverse polarity is blocked by D1, not by Q1 |
| Q4 orientation | PASS | source on `14V`, drain on `14V_SWITCHED`; off state blocks |
| D1, D9 (series protection) | PASS | anode on the supply side, cathode toward the load (netlist pins 2→1 with SOD-128 pin 1 = cathode, as drawn) |
| D2 gate clamp | PASS | cathode on Q1 source, anode on gate: limits VGS to −12 V |
| D8 master pull-up diode | PASS (DNP) | VBAT → anode → cathode → R12‖R13 → LIN |
| D13 SS56B freewheel | PASS | cathode on PH, anode on GND; same library part as the orienter |
| L1 MCZ1210AH winding pairing | PASS | TDK circuit diagram: windings **1–4** and **2–3**. Netlist: L1-1 `USB_CONN_N` ↔ L1-4 `USB_D_N`; L1-2 `USB_CONN_P` ↔ L1-3 `USB_D_P`. Straight through, no transposition. The schematic symbol names its pins "1, 2, 4, 4" — cosmetic, worth fixing in the library |
| MD1 antenna | PASS | DS14096 §3: internal antenna requires pins 6 and 7 shorted; `NetMD1_6` = MD1-6, MD1-7 |
| MD1 pin walk (77 pins) | PASS | all VSS/VSSRF/VSSSMPS on GND; VDDA/VBAT/VDDSMPS on 3V3_D with 100 nF each (C4, C5) as the datasheet asks; NRST with C6; PA13/PA14 to J8; PA0 wake, PA2 switch, PA3/PB5 UART; all other GPIO deliberately open |
| U1 TJA1027 pin walk | PASS | 1 RXD (open-drain, pulled up by R16 — required), 2 SLP_N (R15 up = normal mode), 3 n.c., 4 TXD, 5 GND, 6 LIN, 7 VBAT via D9, 8 n.c., 9 EP to GND |
| U2 / U3 pin walk | PASS except F1 | BST/BOOT caps 0.1 µF / 10 nF on SW/PH; GND and EP tied |
| J7 USB-C | PASS except F4 | both VBUS pads, both GND pads, D+/D− on both orientations (A6/B6, A7/B7), SBU open, shell to CHS_GND |
| J9 OBD-II | PASS | 4 → CHS_GND, 5 → GND, 6/14 CAN, 7 K, 16 → 14V_SWITCHED; 15 (L-line) open |
| J1 header pin use | FAIL (F5) for pins 20/22; otherwise plausible | LIN_TXD → 37 (PC6 = USART6_TX), LIN_RXD → 14 (PC7 = USART6_RX), SWO capture → 16 (PA3 = USART2_RX), 14V_ENABLE → 11 (PK1), BLE_RESET → 36 (PH4), BLE_WKUP → 33 (PJ10). Pins 39/40 (USB_N/USB_P on the header) are on no net; USB goes through J3 instead |

### Section 6 — BOM ↔ netlist ↔ schematic

78 components in the netlist, 69 BOM rows after DNP removal. Values match field by field
except: R10 fitted in BOM but NF on the sheet (F6); Comment/MPN swap on J3/J4/J5 (N3); C26
package text (N4). R12, R13, D8 correctly excluded.

### Section 7 — firmware constraints (no firmware exists yet)

- Tri-state PRODUCT_SW_* before dropping 14 V (N5).
- Drive BLE_RESET open-drain; the module has an internal NRST pull-up and C6.
- 14V_ENABLE idles low through R21 (100K) during dev-board reset, so the DUT stays off — good.
- BLE UART cannot be on PA0_C/PA1_C (F5).
- TJA1027: SLP_N must be driven high (or left to R15) before TXD traffic; RXD is open-drain.

### Section 8 — interfaces

| Item | Verdict | Evidence |
|---|---|---|
| Ground scheme vs the product | PASS | Tester: OBD pin 4 and USB shell → CHS_GND, tied to GND through 10 M ‖ 1 nF 2 kV (R9/C2 and R22/C9 in parallel). Product: its OBD pin 4 and USB shell are also on an RC-isolated CHS_GND, and its pin 5 is on GND. The two boards therefore share ground through OBD pin 5 (and the USB cable ground); pin 4 floats between two RC networks. Acceptable on a bench |
| CAN | NOT CHECKED | tester has no transceiver; J2 carries CAN_H/L from the dev-board CAN board to OBD 6/14 with R1 120 Ω. Product has no termination (its CAN2_P/N nets hold only the transceiver, caps and ESD). Whether the dev-board CAN board adds its own 120 Ω is unknown |
| K-line | PASS (N6) | |
| USB | FAIL (F4), N2 | |
| 14 V switch | PASS | Vgs −14 V; gate 10K up / 1K drive; R18 bleed 2.2K → 1 s to zero with 100 µF of DUT input capacitance; D9 keeps the TJA1027 VBAT off with the rail |
| Input protection | PASS except F2 | D4 TVS, D1 reverse block (−24 V case: D2/D3/Q2/Q3 junctions all conduct through ≥10K — < 3 mA each), 10 ms gate soft-start, no fuse (the 24 V supply's own limit is the fuse — consider a 2 A PTC) |
| Programmers | FAIL (F7) | |

## Not checked

| Item | Why | What it would take to close |
|---|---|---|
| TPS54302 EN threshold, pull-up current, abs max; TPS5450 ENA behaviour | datasheets not on disk; values from memory and from the orienter running | drop both PDFs into `Datasheets/` and re-run the §1 rows |
| H753 dev board: which J1 pins are really PA9/PA10/PA0_C; whether it **supplies** 3.3 V and 5 V on J1 pins 1–4 or **consumes** them; whether its USB port sources VBUS; CAN board termination; J4 "ENABLE P.S." meaning | no dev-board schematic in the repo | add it to `docs/` and resolve F3 against it |
| STM32WB15 AF table for PB5 = LPUART1_TX / PA3 = LPUART1_RX | from memory | check DS of the WB15CCY (the module DS does not carry the AF table) |
| LCSC numbers C17256390, C17393620, C22436137, C21934755 identity | offline | spot-check against LCSC before ordering |
| Electrolytic ESR for the 14 V ripple number | RVT1H221M1010 datasheet absent | N1's fix makes it moot |
| Inrush into the product when Q4 closes | product's +VBAT bulk capacitance unknown | read C11 on the product sheet; if >100 µF consider 10 nF gate-drain on Q4 for a softer edge |
| J1 socket vs the dev board's header (pitch, pin 1, keying) | no dev-board mechanical data | — |
| §9 fabrication | no PcbDoc yet | pre-order pass |
