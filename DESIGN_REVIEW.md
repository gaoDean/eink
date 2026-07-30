# de-link PCB — Design Review

**Board:** `minRead_pcb` (ESP32‑S3 dev hub: 4‑bit SDIO, e‑paper driver, series LED driver, USB‑C power path + single‑cell LiPo charger, two resistor‑ladder button arrays, 12‑pin dev header)
**Reviewed from:** KiCad schematic PDF, netlist, BOM, and `.kicad_pcb` / `.kicad_pro` (KiCad 9).
**Reviewer:** automated design review — treat as a checklist to confirm against the live board, not a substitute for a fab DRC/DFM run.

---

## Bottom line

The schematic is well thought through and the topology is sound — the power path, DW01A/FS8205A cell protection, MCP73832 charger, ESP32‑S3 native USB, externally‑pulled‑up 4‑bit SDIO, and the LED/e‑paper boost rails are all reasonable and the designer's own annotations show the intent clearly. The ESD strategy is good on most exposed ports.

The biggest risk to your three stated goals (power integrity, damage safety, and passing EMI/EMC) is **physical, not schematic: this is a 2‑layer board carrying native USB, ~26 MHz SDIO, and three switching converters.** That combination is the single hardest thing to get through a formal EMC scan on two layers. Everything else below is comparatively minor.

Severity key: 🔴 will likely bite you · 🟡 worth fixing before fab · 🟢 polish / verify.

---

## 1. Stackup, power distribution & EMC  (your "efficiently power distributed" + "pass EMI/EMC" goals)

🔴 **2‑layer stack with three switchers + USB + SDIO.** The board is `F.Cu`/`B.Cu` only. You have a GND pour on both layers and a 3V3 pour on top, which is the right instinct, but the top "ground plane" is cut by ~496 routed segments, so it is not a continuous reference. For radiated‑emissions and immunity testing, the return path under your fast edges (USB D±, SD_CLK, the boost SW nodes) is what determines whether you pass. **Strongest recommendation: move to a 4‑layer stack (Sig / GND / PWR / Sig).** A solid uninterrupted GND plane on layer 2 directly under the signal layer is the highest‑leverage change you can make for EMC and is usually only a few dollars more at any fab. If you must stay 2‑layer, treat one entire side as ground and route *everything* on the other, accepting jumpers — do not let signal traces carve up the ground side.

🔴 **Switching converters are the emitters to control.** You have three: the LED boost (LM27313 + 22 µH `L2`, up to 29 V on `LED_SW`), the e‑paper charge‑pump boost (BSS138 + 22 µH `L1`), and the buck/charger path. Each switching loop (input cap → inductor → diode → output cap, plus the catch/return) must be a tiny tight loop with its return directly beneath it. On 2 layers this is the part most likely to radiate. Verify: input/output ceramics placed hard against the IC pins, the SW node kept physically small (it's the hot antenna), and a local ground stitch right at each converter.

🟡 **Single net class for everything.** The project has only the `Default` class (0.2 mm track, 0.15 mm clearance, 0.6/0.3 via). Power nets are getting the same 0.2 mm default as signals except where you manually widened them (the layout uses a mix of 0.2/0.4/0.6/1.0/1.3 mm). 0.2 mm of 1 oz copper is ~0.5 A; for VBUS, BAT, the LED boost output and 3V3 main feed that's marginal. Create explicit net classes: `Power` (≥0.5 mm, wider for VBUS/BAT/LED_SW), `USB` (90 Ω diff pair), and `HV` for the e‑paper rails, and assign them so widths/clearances are enforced rather than hand‑drawn.

🟡 **Star/local decoupling looks good, confirm placement.** BOM shows generous bulk + 0.1 µF/1 µF/4.7 µF local caps and a 22 µF + 0.1 µF pair on the ESP32 3V3. Make sure each decap is physically next to its pin with a short via to plane, not just net‑connected — on 2 layers the via inductance dominates.

🟢 **High‑voltage e‑paper rails (VGH/VGL/VDH/VDL, ±~15–22 V).** Give these extra clearance (your 0.15 mm min is fine electrically but tight for >20 V swings near other nets) and keep them away from the USB/SD edges. Confirm the charge‑pump Schottkys (MBR0530) and caps are rated with margin for the actual peak rail.

---

## 2. ESD protection  (your SD / USB / dev‑header "ESD safe" goal)

🟢 **USB‑C data + CC:** `TPD4E1U06DBVR` array on the data lines plus CC handling — good. CC1/CC2 have the correct 5.1 k pull‑downs (`R2`,`R3`) for a UFP/sink. ✔
🟢 **Dev header (J6, 12‑pin):** well covered — two `TPD4E1U06DBVR` arrays on the eight GPIO/UART pins, `SMAJ30A` and `PESD2IVN‑UX` TVS on the LED switch / strip lines, and `TSD05CDYFR` on the 3V3 pin. This is the most thoroughly protected port. ✔

🟡 **microSD: CLK and CMD are not ESD‑protected.** The SD slot uses one `TPD4E1U06DBVR` (4 channels) on `DAT0–DAT3` only. `SD_CMD` and `SD_CLK` go to the card connector with series resistors but no clamp. A microSD socket is a user‑touchable, hot‑pluggable port, so for a true "SD ESD‑safe" claim add a second clamp (e.g. a 2‑channel TVS or a 6‑channel array) covering CMD and CLK. The series resistors help with edge rate but don't provide ESD clamping.

🟢 **Placement matters as much as the part.** For all three ports, the TVS/array must sit physically at the connector, before any stub, with a very short, low‑inductance path to ground — otherwise the clamp is bypassed by the connector's own inductance during a strike. Confirm in layout.

---

## 3. Charger, power path & battery safety  (your "safe from damage" goal)

🟢 **Cell protection:** `DW01A` + `FS8205A` gives over‑charge / over‑discharge / over‑current cutoff on the cell — correct classic single‑cell protection, plus a reverse‑polarity P‑FET (`AO3401`) on the battery connector. Good defense in depth for an unprotected cell. ✔
🟢 **Charger:** `MCP73832‑2‑OT`, `PROG` ≈ 10 k → ~100 mA charge. Safe and gentle; just confirm it matches your actual cell's C‑rate (100 mA suits packs ≥ ~200 mAh). ✔
🟢 **Power path:** P‑FET ideal‑diode‑style USB/battery mux feeding the `AP2112K‑3.3` LDO — fine for this current class. ✔

🟡 **MCP73832 thermal.** The MCP73832 dissipates `(VBUS − VBAT) × Ichg` in a tiny SOT‑23‑5. At 100 mA it's modest, but if you ever raise `PROG` current, give it a ground‑plane copper pour for heat. Worth confirming the thermal pad/pour now.

🟡 **No explicit VBUS over‑voltage clamp on the 5 V rail itself.** You have ESD on the data/CC lines; confirm there is a TVS on VBUS (the `TSD05CDYFR` appears on a 3V3/IO node, not the raw 5 V input). A 5 V‑working TVS across VBUS→GND right at J1 protects the charger and power‑path FETs from hot‑plug spikes and bad chargers. (Verify — I couldn't confirm a VBUS‑rail clamp from the netlist.)

🟢 **PWR_FLAG / DRC hygiene:** PWR_FLAGs are present on the supply nets, which means you've been running ERC. Good — make sure ERC is clean (no implicit power, no unconnected) before fab.

---

## 4. Signal integrity & functional items

🔴 **Battery‑monitor ADC source impedance is too high.** The divider is `R10`=1 M / `R12`=1 M (≈500 kΩ source). The ESP32‑S3 SAR ADC wants a source impedance on the order of ≤10 kΩ for accurate sampling; 500 kΩ will give slow settling and offset error. The 1 µF `C8` to ground helps a lot for *slow* DC battery reads (it holds charge during the sampling window), so it may be acceptable in practice — but characterize it. Safer fix: drop to ~100 k/100 k (still only ~16 µA idle from a charged cell) and keep the cap, or buffer with a tiny op‑amp. Also note the ESP32 ADC nonlinearity — plan to use the eFuse calibration.

🟡 **USB D± as a real differential pair.** Native USB on the S3 is only Full‑Speed, so it's forgiving, but still route D+/D− as a length‑matched ~90 Ω pair, tightly coupled, over continuous ground, away from the switchers. Right now there's no USB net class, so the pair geometry isn't being enforced.

🟡 **SDIO at 4‑bit/high speed.** You did the right thing with external pull‑ups on DAT0–3 + CMD and series resistors — good for signal integrity and for letting the bus idle high. Keep `SD_CLK` short, route the six SD lines as a tight group over ground, and length‑match loosely. The series Rs are also your main edge‑rate (EMI) control here, so don't set them to 0 Ω; tune to ~22–33 Ω.

🟡 **Resistor‑ladder button arrays — confirm a defined idle level.** Make sure each ADC ladder has a defined resting voltage (a pull so "no button" reads a stable rail, not a float) and that the per‑button steps are far enough apart to be unambiguous after ADC noise/nonlinearity. With the S3 ADC's ~±I errors, leave generous margin between codes, and debounce in firmware.

🟢 **E‑paper temperature sensor not connected.** `TSCL`/`TSDA` (panel temp‑sensor I²C on the 24‑pin ZIF) don't appear in the netlist — they're unconnected. Many e‑paper panels need the host to read temperature to pick the right refresh waveform; if your panel relies on external temp read, image quality will drift with temperature. Confirm your specific panel uses its *internal* sensor; if not, wire TSCL/TSDA to two spare GPIO.

🟢 **Unused GPIO header pins.** Several `UNUSED_GPIO_*` nets are broken out to J6 — fine, but make sure none are strapping pins (IO0, IO45, IO46 on the S3 affect boot/flash voltage) left able to float into a bad boot state. IO45/IO46 in particular should have defined levels.

---

## 5. Pre‑fab checklist

- Run **DRC** with the real fab's constraints (your `min_track_width` is set to 0.0 — set it to the fab minimum so DRC actually catches hairlines).
- Run **ERC** clean; confirm every PWR_FLAG/no‑connect is intentional.
- Confirm **courtyard/clearance** on the USB‑C, microSD, ZIF and the two FH34SR connectors — fine‑pitch parts are the usual DFM rejects.
- Verify **edge clearance** (you have 0.475 mm copper‑to‑edge — good) around the board outline and mounting holes.
- Decide mounting‑hole grounding: all six H‑pads tie to GND. That's good for chassis/EMC *if* the enclosure is conductive and you want HF bonding; if it's plastic it's harmless. Just be aware it can form ground loops in some setups.
- Generate and eyeball the **3D view / fab layers** for silk over pads, missing paste, tented vias near connectors.

---

## What I'd prioritize, in order

1. **Re‑spin to 4 layers with a solid GND plane** (biggest EMC win, fixes most return‑path problems at once).
2. **Add ESD clamp on SD CLK/CMD**, and confirm a **VBUS TVS** at J1.
3. **Fix the battery‑monitor divider impedance** (or validate the cap‑assisted read empirically).
4. **Add real net classes** (Power / USB / HV) and verify switcher loop layout + decap placement.
5. Tidy the smaller items: strapping‑pin levels, e‑paper temp sensor, ladder idle levels.

*This review is based on static analysis of the design files. It can't measure actual trace routing quality, return paths, or loop areas — those need a look at the layout itself or an EMC pre‑scan. I'd be glad to go deeper on any single block (e.g. dump the exact routing/loop for the LED boost, or check every strapping pin's net) if you want.*
