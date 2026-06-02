# Change-set — FAD batch 02 (Chapter 2.1 review)

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Scope:** FAD §2.1 review feedback (+ two consistency companions in §2/§2.2). Anchors are byte-exact.
**Apply method:** `str_replace` / exact find-replace. Do not normalise whitespace or the `→`, `×`, `~`, `²` glyphs.
**Pre-flight:** apply **after** batch 01. Confirm `arch/fad/FAD.md` matches the reviewed v0.7 line state.

Tags: **[CONFIRMED]** = Marco approved explicitly. **[RECOMMENDED]** = source-backed, not explicitly confirmed — drop the hunk if undesired.

Key decision on the blocker: 19.6608 Gbps is the implemented GT line rate (QPLL1: REFCLK 307.2 MHz ×N=32 → VCO 9.8304 GHz ×2), canonical per **§4.2.2 / ADR-0010**. Parent docs (SFU-001 §6.1 / ARCH-001 §12) still state 19.2 Gbps nominal and will be aligned; divergence is marked `[ASSUMPTION … expiry]` so it stays greppable. Citation deliberately points at §4.2.2, **not** SFU-001, to avoid attributing 19.6608 to a source that doesn't yet carry it.

Suggested commit: single `docs(fad): chapter-2.1 review fixes (Aurora rate, SFP28 breakout, UL beacon fold, RF BW)`.

---

## C1 — §2.1 OBG interface: Aurora rate + active SFP28 breakout  ·  blocker + breakout-high  ·  **[CONFIRMED]**  ·  `arch/fad/FAD.md`

Both fixes live on one sentence, so applied as one hunk. Rate 19.2→19.6608 (+ per-direction aggregate); LC-duplex→active SFP28; citation retargeted; parent-doc divergence marked.

**FIND:**
```
Q0 QSFP28 is the external bidirectional optical interface carrying four Aurora 64B/66B lanes at 19.2 Gbps each between the optical matrix and the SFU. A QSFP28-to-4×LC duplex breakout cable fans out the four lanes onto separate Polatis 6000i matrix ports. The Q0 link is full-duplex: the same physical lanes carry DL bin data (DCU → SFU) and UL bin data (SFU → DCU) simultaneously. (ARCH-001 §12.1; SFU-001 §9.1)
```

**REPLACE:**
```
Q0 QSFP28 is the external bidirectional optical interface carrying four Aurora 64B/66B lanes at 19.6608 Gbps each (aggregate 78.6432 Gbps per direction) between the optical matrix and the SFU. The QSFP28 cage is populated with a 1:4 active SFP28 transceiver breakout (QSFP28 → 4× SFP28), fanning the four lanes onto separate Polatis 6000i matrix ports. The Q0 link is full-duplex: the same physical lanes carry DL bin data (DCU → SFU) and UL bin data (SFU → DCU) simultaneously. (Line rate per §4.2.2 / ADR-0010; breakout per SFU-001 §6.1, OI-S04; ARCH-001 §12.1.) [ASSUMPTION: 19.6608 Gbps is the implemented GT line rate; SFU-001 §6.1 / ARCH-001 §12 still state 19.2 Gbps nominal — expiry: parent-doc line-rate alignment.]
```

---

## C2 — §2.1 UL path: fold in UL beacon / OBG control-channel duplication  ·  high  ·  **[CONFIRMED]**  ·  `arch/fad/FAD.md`

States the beacon extraction/duplication is folded into `bins_sel_ul` and expanded in §2.2/§7.6 (per Marco). Citation §7.1–§7.5 → §7.1–§7.6.

**FIND:**
```
Those bins are routed by `bins_sel_ul` into OBG Aurora output frames for return to the DCU via `obg_link`. The Q0 port is full-duplex, so the same physical QSFP28 breakout carries DL and UL bin traffic in opposite directions simultaneously. (SFU-001 §7.1–§7.5; ARCH-001 §12.1)
```

**REPLACE:**
```
Those bins are routed by `bins_sel_ul` into OBG Aurora output frames for return to the DCU via `obg_link`. `bins_sel_ul` also extracts this SFU's assigned UL frequency-correction beacon and duplicates it as a ~10 MHz / ~120-bin (ASIC-grid) control beam onto the OBG control channel of all four egress lanes; this function is folded into `bins_sel_ul` here and expanded in §2.2 and §7.6. The Q0 port is full-duplex, so the same physical QSFP28 breakout carries DL and UL bin traffic in opposite directions simultaneously. (SFU-001 §7.1–§7.6; ARCH-001 §12.1)
```

---

## C3 — §2.1 DL path: RF output bandwidth  ·  high  ·  **[CONFIRMED]**  ·  `arch/fad/FAD.md`

**FIND:**
```
Together, the two sub-bands support the system-specified 1,600 MHz occupied RF output bandwidth.
```

**REPLACE:**
```
Together, the two sub-bands provide ~1,638.4 MHz aggregate usable occupied RF output bandwidth (~819.2 MHz per sub-band); "1.6 GHz" is a nominal shorthand.
```

---

## C4 — §2.2 RF bandwidth (consistency companion)  ·  medium  ·  **[RECOMMENDED]**  ·  `arch/fad/FAD.md`

Same `1,600 MHz` wording recurs in §2.2; align so §2 is internally consistent. Drop if you want to handle §2.2 in its own batch.

**FIND:**
```
Together the two sub-bands produce the 1,600 MHz occupied RF bandwidth.
```

**REPLACE:**
```
Together the two sub-bands provide ~1,638.4 MHz aggregate usable occupied RF bandwidth (~819.2 MHz per sub-band; "1.6 GHz" nominal shorthand).
```

---

## C5 — Figure 2.1 caption: ASIC-nominal qualifier  ·  medium  ·  **[RECOMMENDED]**  ·  `arch/fad/FAD.md`

Cannot edit the PNG; caption note qualifies all numeric labels in the figure. (Prose `4,096 Msps` instances are already followed by the FPGA-scaled value at the "Shared RF and RFDC resources" paragraph, so they need no change.)

**FIND:**
```
*Figure 2.1 — SFU top-level bidirectional datapath. DL flows left→right (upper lane); UL flows right→left (lower lane). Shared blocks shown once at centre. Source: `arch/fad/diagrams/top_datapath.drawio`.*
```

**REPLACE:**
```
*Figure 2.1 — SFU top-level bidirectional datapath. DL flows left→right (upper lane); UL flows right→left (lower lane). Shared blocks shown once at centre. Numeric labels are ASIC-satellite nominal; FPGA-satellite values scale ×1.024 (§4.0). Source: `arch/fad/diagrams/top_datapath.drawio`.*
```

---

## C6 — §2 `.drawio` authority wording  ·  low  ·  **[RECOMMENDED]**  ·  `arch/fad/FAD.md`

Removes the "source of truth" reading that could be taken as diagrams overriding prose/§6.

**FIND:**
```
Authoritative diagram source files (`.drawio`) and rendered PNGs live in `arch/fad/diagrams/`. The figures below are embedded PNG renders; the `.drawio` files are the editable source of truth.
```

**REPLACE:**
```
Editable diagram source files (`.drawio`) and rendered PNGs live in `arch/fad/diagrams/`. The figures below are embedded PNG renders; the `.drawio` files are the editable source for the rendered figures. FAD text and the §6 module inventory remain authoritative.
```

---

## C7 — OPTIONAL housekeeping: change-log row  ·  `arch/fad/FAD.md`

Apply only if bumping the version this batch (set the number to follow whatever batch 01 left; default below assumes batch-01 C13 landed at 0.7.1). Marco may rewrite the prose.

Add immediately under the latest change-log row in §15:
```
| 0.7.2 | 2026-05-28 | Marco Pausini | **Chapter-2.1 review fixes (pre-G1).** §2.1 Aurora line rate corrected 19.2 → 19.6608 Gbps (aggregate 78.6432 Gbps/direction), cited to §4.2.2/ADR-0010 with a parent-doc-divergence [ASSUMPTION] marker (SFU-001/ARCH-001 still nominal 19.2, alignment pending). SFU-side breakout updated LC-duplex → 1:4 active SFP28 (OI-S04). UL frequency-correction beacon / OBG control-channel duplication folded into `bins_sel_ul` in §2.1 with pointer to §2.2/§7.6. RF output BW reworded to ~1,638.4 MHz aggregate usable (~819.2 MHz/sub-band) in §2.1 and §2.2. Figure 2.1 caption flagged ASIC-nominal (FPGA ×1.024). `.drawio` authority wording clarified (FAD text + §6 inventory authoritative). |
```

---

## Post-apply check

```sh
grep -n "19.6608 Gbps each" arch/fad/FAD.md          # C1 present
grep -c "19.2 Gbps" arch/fad/FAD.md                  # expect 0 in §2.x prose (only inside ASSUMPTION marker)
grep -n "active SFP28 transceiver breakout" arch/fad/FAD.md   # C1 breakout
grep -n "4×LC duplex\|QSFP28-to-4×LC" arch/fad/FAD.md         # expect 0
grep -n "folded into \`bins_sel_ul\` here and expanded" arch/fad/FAD.md  # C2
grep -c "1,600 MHz" arch/fad/FAD.md                  # expect 0 (both §2.1 + §2.2 fixed) if C4 applied; else 1
grep -n "1,638.4 MHz" arch/fad/FAD.md                # C3 (+C4)
grep -n "ASIC-satellite nominal; FPGA-satellite values scale" arch/fad/FAD.md  # C5
grep -n "editable source of truth" arch/fad/FAD.md   # expect 0 if C6 applied
```

Render §2.1/§2.2 and eyeball the corrected sentences.
