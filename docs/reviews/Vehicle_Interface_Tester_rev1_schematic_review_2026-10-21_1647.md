# Vehicle Interface Tester rev1 schematic review — 2026-10-21 16:47 (second pass)

Inputs: schematic PDF 16:46, Protel netlist and BOM 16:47, from the working tree (sheets saved
16:36–16:46; `5V_Buck.SchDoc` and `USB_Host_Power.SchDoc` deleted, `3V3_Buck.SchDoc` new; nothing
committed since `2264ae9`). Same datasheet set as the first pass; TPS5450/TPS54302 still absent.
Firmware: none. Checklist: `orienter_PCB/docs/hw_review_checklist.md` §1–§8 (schematic only).

Method as before: every connection taken from the netlist, values from the BOM, symbol pin names
from the `.SchDoc` records; rendered sheets used only to read intent.

## What changed since the 14:58 pass

| First-pass item | Status now |
|---|---|
| F1 14 V divider inverted | **FIXED** — R23 21K on `14V`, R24 2K to GND: 1.221 × (1 + 21/2) = **14.04 V** |
| F2 OV cut-off at ~21 V | **FIXED, with one new defect (G1 below)** — TL431 stage in: R2 24.9K / R4 2.49K → **27.4 V**, C2 10 nF, cathode to Q2 base, four anodes on GND; PTC1 2 A ahead of the TVS; net renamed `24V_SW` |
| F3 unsourced rails | **PARTLY** — `24V_SW` now feeds J4 (dev board) and U3; the 5 V buck is gone, replaced by a 3.3 V buck that feeds the module, J8, J11 VTref and the LED. Still floating: `+5V`, `VCC` (deliberate, pending the dev board), `3V3_D` (new orphan, G3), and the 3.3 V buck's own input (G2) |
| F4 USB-C Rd instead of Rp | **HALF** — R7 is now 56K to VBUS on CC2, but R6 is still 5.1K to GND on CC1 (G4) |
| F5 BLE UART on PA0_C/PA1_C | **MOVED** to pins 25/27 (PA9/PA10) — but TX→TX, RX→RX (G5) |
| F6 BOOT0 1K/1K, R9 NF only as text | **UNCHANGED** — BOM still orders R9 (now R9/R10) |
| F7 J11 VTref NC | **FIXED** — J11-1 on `3V3` (tester's rail, fine: same level as the dev board) |
| N1 no ceramic on 14 V | unchanged (minor) |
| N2 VBUS switch / HOST_DET | unchanged, pending dev-board test |
| N3 J3/J4/J5 Comment swap, N4 C17 package text | unchanged (cosmetic) |

## Headline — five gates before layout

| | |
|---|---|
| **G1** | **D4 (BAT54W) is installed forwards.** Cathode (pin 1) on GND, anode (pin 2) on REF. It clamps REF at ≈0.3 V, so the TL431 can never reach 2.495 V and **the over-voltage cut-off is disabled**; R2 also dumps ≈1 mA through the diode continuously. Flip it: cathode on REF, anode on GND. |
| **G2** | **The 3.3 V buck has no input.** F1-2 is on the buck's VIN node; **F1-1 is on no net**. The sheet's left-hand "24V" is a port with nothing connected on the parent sheet. Tie it to `24V_SW`. Until then the module, the LIN pull-ups, J8 and J11 are dead. |
| **G3** | **`3V3_D` is now an orphan**: only R13, R14, R15 (LIN TXD/SLP_N/RXD pull-ups) are on it. The module moved to `3V3`; K-Line.SchDoc did not. Rename the three pull-ups to `3V3`. |
| **G4** | **CC1 still has Rd.** R6 = 5.1K to GND on CC1; R7 = 56K to VBUS on CC2. A cable can land on either orientation, so both need 56K to VBUS. |
| **G5** | **BLE UART is not crossed.** `BLE_TX` (module PB5, LPUART1_TX) → J1-25 = **PA9 = USART1_TX**; `BLE_RX` (module PA3, LPUART1_RX) → J1-27 = **PA10 = USART1_RX**. Transmitter into transmitter. Swap the two wires on the dev-board sheet (or rely on the H7 USART `SWAP` bit, which works but is a trap for the next person). |

Plus F6 carried: BOOT0 is still 1.65 V as ordered.

---

## Findings

| # | Section | Item | Verdict | Number / reasoning | Fix |
|---|---------|------|---------|--------------------|-----|
| **G1** | 5 | **D4 BAT54W orientation** | **FAIL** | Symbol pins from the SchDoc: pin 1 = Cathode, pin 2 = Anode. Netlist: `GND` holds **D4-1**, `NetC2_1` (REF) holds **D4-2**. Anode on REF, cathode on GND = forward-biased in normal operation. REF cannot exceed V_F ≈ 0.3–0.4 V, TL431 never trips, and I(R2) = (24 − 0.35)/24.9K = 0.95 mA flows through D4 permanently. | Rotate D4: cathode (pin 1) to REF, anode (pin 2) to GND. Then re-read the BAT54W vendor drawing to confirm pin 1 = cathode on the ordered part (LCSC C699107). |
| **G2** | 8 | **3.3 V buck input open** | **FAIL** | `NetC19_1` = C19, C20, C21, C22, **F1-2**, R26-2, U4-3. **F1-1 appears in no net.** On the rendered sheet the fuse's left end goes to a harness/port symbol "24V" that the parent `Power_Supplies.SchDoc` does not connect. | Replace the port with the `24V_SW` power port (as on the 14 V sheet) or wire it on the parent. |
| **G3** | 8 | **`3V3_D` orphan** | **FAIL** | `3V3_D` = {R13-2, R14-2, R15-2}. No source. TJA1027 SLP_N therefore floats low through its internal 250K pull-down → transceiver stays in **sleep**; TXD floats low through 125K → **permanent dominant** on K-line; RXD open-drain never rises. | Change the three power ports on K-Line.SchDoc to `3V3`. |
| **G4** | 8 | **USB-C CC1** | **FAIL** | `NetJ7_A5` = J7-A5 (CC1), R6-1; R6-2 on GND; R6 = 5.1K. `NetJ7_B5` = J7-B5 (CC2), R7-2; R7-1 on `VBUS`; R7 = 56K. One side Rd, one side Rp. | R6 → 56K to `VBUS`. Delete the GND connection. |
| **G5** | 5, 7 | **BLE UART TX/RX crossing** | **FAIL** | `BLE_TX` = MD1-26 (PB5 = LPUART1_TX on WB15) and J1-25 (sheet label PA9 = USART1_TX on H7). `BLE_RX` = MD1-28 (PA3 = LPUART1_RX) and J1-27 (PA10 = USART1_RX). Both ends drive the same wire. | Swap: BLE_TX → J1-27 (PA10), BLE_RX → J1-25 (PA9). Confirm the header's 25/27 labels on the real dev board. |
| **F6** | 5, 6 | **BOOT0 1K/1K** | **FAIL (carried)** | `NetMD1_22` = MD1-22, R9-1, R10-1; R9-2 on `3V3`, R10-2 on GND; BOM row "R9, R10, R18 \| 1K \| 3". BOOT0 = 1.65 V. | Set R9's fitted/DNP parameter so the BOM drops it. |
| — | 2 | **14 V divider** | **PASS** | R23 (21K) R23-2 on `14V`, R23-1 on VSENSE; R24 (2K) R24-1 on GND. 1.221 × 11.5 = **14.04 V**. Sheet note matches. | — |
| — | 2 | **3.3 V divider** | **PASS** | R27 68K on `3V3` / R29 15K to GND: 0.596 × (1 + 68/15) = **3.298 V**. C24 47 pF across R27 (feed-forward zero ≈ 50 kHz). | — |
| — | 1, 2 | **OV threshold (once G1 is fixed)** | **PASS** | 2.495 × (1 + 24.9/2.49) = **27.4 V**; ±1 % ref, ±1 % R → 27.0–27.8 V; I_REF 2 µA × 24.9K = +50 mV. Clears 24 V + 5 % by 1.8 V worst case; under TPS54302 28 V op-max; SMAJ26CA breakdown ≥ 28.9 V so the cut-off acts first. Cathode current when tripped (24 − 0.7 − 2.5)/10K = 2.1 mA ≥ 1 mA min. | — |
| — | 1 | **TL431 pin walk** | **PASS** | 1 CATHODE → `NetQ2_1` (Q2 base, R3); 8 REF → `NetC2_1` (R2, R4, C2, D4); 2/3/6/7 ANODE → GND; 4/5 NC. | — |
| — | 8 | **PTC1 / TVS order** | **PASS** | J6-1 → PTC1-1; PTC1-2 → `NetC1_1` with D3 (TVS), Q1 source, R2, R3, D2, C1. PTC ahead of the TVS, so the PTC limits what D3 absorbs. 2 A hold vs < 1 A draw. | — |
| — | 3, 4 | **3.3 V buck operating point** | **PASS** | ΔIL = 3.3 × (1 − 3.3/24)/(10 µ × 400 k) = **0.71 A**, L3 5.5 A. Duty 13.75 % → on-time 344 ns at 400 kHz, above the part's ≈100 ns minimum. UVLO unchanged (≈17.6 V, estimate). Load is the module (< 50 mA) + pull-ups + LED (1.9 mA): light-load Eco-mode territory, fine. Output 44 µF ceramic + 220 µF, same structure the orienter runs. | — |
| — | 1 | **J11 VTref on the tester's 3V3** | **PASS, noted** | J11-1 → `3V3`. Same logic level as the dev board, so ST-Link is satisfied. It will also report "target connected" when only the tester is powered — cosmetic. | Optionally move to J5-1 later if the dev board exposes its own 3V3. |
| — | 4 | **New LED** | **PASS** | D13 on 3V3 through R25 680R: (3.3 − 2)/680 = 1.9 mA. | — |
| — | 8 | **J4 fed from `24V_SW`** | **PASS** | J4-1/2/3 on `24V_SW`; J4-4 "ENABLE P.S." still NC — unknown until the dev board arrives. | — |
| — | 6 | **New parts in BOM** | **PASS** | TL431AIDR C6970, BAT54W C699107, PTC 2 A 1210 C20808, fuse 1.5 A C2936103, 56K C25796, 24.9K C25962, 2.49K C22908, 68K C36871, 15K C25756 all present with LCSC numbers (identities not verified offline). R11/R12/D8 still correctly absent (DNP). | — |
| — | 5 | **Unchanged and still PASS** | PASS | Q1/Q3 orientation, D1/D9 polarity, D2 clamp, D12 freewheel, L1 pairing (1–4, 2–3), module antenna short, module and TJA1027 pin walks, OBD/CHS_GND scheme, 14 V switch (now Q3/Q4), R17 bleed 89 mW, CAN 120R. | — |

## Carried, minor

N1 (10 µF ceramic on 14 V), N2 (VBUS switch + HOST_DET), N3 (PicoBlade Comment/MPN swap on
J3/J4/J5), N4 (C17 package text), N5 (firmware must tri-state PRODUCT_SW_* before dropping 14 V,
and now also: TJA1027 SLP_N high before TXD traffic).

## Not checked

Same list as the first pass. Added: J4-4 "ENABLE P.S." meaning; whether the dev board accepts
`24V_SW` on VDD_IN while `+5V`/`VCC` are left open (pending the bench test Jared planned).
