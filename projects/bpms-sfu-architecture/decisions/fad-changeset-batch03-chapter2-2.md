# Change-set — FAD batch 03 (Chapter 2.2 review)

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Scope:** FAD §2.2 review + the updated `subband_processing.drawio` (delivered alongside; re-render its PNG).
**Apply method:** `str_replace` exact find-replace. Preserve `→ × ± ~ §` glyphs and backticks.
**Pre-flight:** apply after batches 01–02. **B2 overlaps batch-02 C4** — read the B2 note before applying.

Resolutions: blocker (beacon insertion point) resolved via source hierarchy — ARCH-001 SYS-GEN-07 ("after Band Gain") is **Open under OI-B19, which delegates the SFU-side injection point to SFU-001 §6.8**; FAD follows §6.8 (post-synthesis, pre-Band-Gain, pre-Band-Doppler). Captured as a source erratum, not a placement change. cw_beacon scope: **single per-card** (SFU §6.8 "anywhere across both sub-bands"; §6 inventory = 1 instance). Bin count corrected 9,600 → ~9,830 (80% of 12,288 frame; tied to §6.7 usable BW), `[INFERRED]`-tagged.

Two artefacts in this batch: **(a)** this change-set (FAD prose); **(b)** `subband_processing.drawio` — replace the repo copy at `arch/fad/diagrams/subband_processing.drawio` and re-render `subband_processing.png`.

---

## B1 — §2.2 `cw_beacon` prose: single-per-card scope + insertion-point erratum  ·  blocker + high  ·  `arch/fad/FAD.md`

Covers the blocker (erratum) and the cw_beacon-scope high item in one hunk (same paragraph).

**FIND:**
```
`cw_beacon` is DL-only. It injects a continuous-wave tone into the DL path after filter-bank synthesis and before Band Gain. The beacon's primary purpose is to provide a reference tone for the satellite CPBF to perform residual frequency correction; secondary uses include Band Gain reference and link verification. The beacon tone is placeable at any frequency within the full available spectrum across both sub-bands. (SFU-001 §6.8)
```

**REPLACE:**
```
`cw_beacon` is a single per-card DL-only block (one instance — not replicated per sub-band). It injects a continuous-wave tone into the DL path after filter-bank synthesis and before Band Gain, so the tone passes through both Band Gain and Band Doppler before reaching `rf_port_sel`. The NCO is tunable anywhere across both sub-bands' spectrum, and the tone is routed into the sub-band carrying the configured frequency. The beacon's primary purpose is to provide a reference tone for the satellite CPBF to perform residual frequency correction — passing through Band Doppler is required so the satellite measures the *residual* offset rather than the full Doppler; secondary uses include the Band Gain closed-loop reference (received beacon power) and link verification. (SFU-001 §6.8) [Source erratum: ARCH-001 SYS-GEN-07 states the DL beacon is injected "after Band Gain"; SYS-GEN-07 is Open under OI-B19, which delegates the SFU-side injection point to SFU-001. FAD follows SFU-001 §6.8 (post-synthesis, pre-Band-Gain, pre-Band-Doppler) — expiry: ARCH SYS-GEN-07 closure / OI-B19.]
```

---

## B2 — §2.2 opening paragraph: RF bandwidth + occupied bin count  ·  high (+ bin-count cleanup)  ·  `arch/fad/FAD.md`

**Overlap with batch-02 C4:** both touch this line. **Apply B2 *instead of* C4** (B2 is the superset: BW + bin counts). If C4 was already applied, its FIND below won't match — use **B2-alt**.

**FIND (primary — original v0.7 line):**
```
This diagram expands the `DL sub-band proc ×2` and `UL sub-band proc ×2` blocks from the top-level datapath (§2.1). The SFU architecture defines two sub-bands, each covering 1,024 MHz native bandwidth for ASIC satellite operation (1,048.576 MHz for FPGA satellite), with approximately 819.2 MHz usable occupied bandwidth after filtering. Together the two sub-bands produce the 1,600 MHz occupied RF bandwidth. Each sub-band has 12,288 total bins (3 parallel lanes × 4,096-point FFT per lane), of which up to 9,600 are occupied (19,200 RF bins per SFU / 2 sub-bands). (SFU-001 §5, §6.4, §12.1; ARCH-001 §5.2)
```

**REPLACE:**
```
This diagram expands the `DL sub-band proc ×2` and `UL sub-band proc ×2` blocks from the top-level datapath (§2.1). The SFU architecture defines two sub-bands, each covering 1,024 MHz native bandwidth for ASIC satellite operation (1,048.576 MHz for FPGA satellite), with approximately 819.2 MHz usable occupied bandwidth after filtering. Together the two sub-bands provide ~1,638.4 MHz aggregate usable occupied RF bandwidth (~819.2 MHz per sub-band; "1.6 GHz" nominal shorthand). Each sub-band has 12,288 total bins (3 parallel lanes × 4,096-point FFT per lane), of which up to ~9,830 are occupied — [INFERRED from §6.7 usable BW / §6.4 frame: 80% of 12,288 = 9,830.4 bins/sub-band → ~19,661 bins/SFU; bin count is invariant across satellite type, only bin width scales ×1.024]. (SFU-001 §5, §6.4, §6.7, §12.1; ARCH-001 §5.2)
```

**B2-alt FIND (only if batch-02 C4 already landed — fixes the bin counts that C4 left untouched):**
```
Together the two sub-bands provide ~1,638.4 MHz aggregate usable occupied RF bandwidth (~819.2 MHz per sub-band; "1.6 GHz" nominal shorthand). Each sub-band has 12,288 total bins (3 parallel lanes × 4,096-point FFT per lane), of which up to 9,600 are occupied (19,200 RF bins per SFU / 2 sub-bands).
```
**B2-alt REPLACE:**
```
Together the two sub-bands provide ~1,638.4 MHz aggregate usable occupied RF bandwidth (~819.2 MHz per sub-band; "1.6 GHz" nominal shorthand). Each sub-band has 12,288 total bins (3 parallel lanes × 4,096-point FFT per lane), of which up to ~9,830 are occupied — [INFERRED from §6.7 usable BW / §6.4 frame: 80% of 12,288 = 9,830.4 bins/sub-band → ~19,661 bins/SFU; bin count is invariant across satellite type, only bin width scales ×1.024].
```

---

## B3 — Figure 2.2 caption: replication scope + ASIC-nominal qualifier  ·  high + medium  ·  `arch/fad/FAD.md`

Aligns the caption with the updated diagram subtitle.

**FIND:**
```
*Figure 2.2 — SFU sub-band processing detail. One representative lane per sub-band is shown; the same structure is instantiated for sub-band A (lower) and sub-band B (upper). Source: `arch/fad/diagrams/subband_processing.drawio`.*
```

**REPLACE:**
```
*Figure 2.2 — SFU sub-band processing detail. One representative sub-band chain is shown; the DL and UL chains are instantiated ×2 (sub-band A lower, B upper). `cw_beacon` is a single per-card block (not replicated). Numeric labels are ASIC-satellite nominal; FPGA-satellite scales BW / Fs / Doppler ×1.024 (bin counts invariant) (§4.0). Source: `arch/fad/diagrams/subband_processing.drawio`.*
```

---

## B4 — diagram asset  ·  `arch/fad/diagrams/`

- Replace `arch/fad/diagrams/subband_processing.drawio` with the updated file delivered in this batch.
- Re-render `arch/fad/diagrams/subband_processing.png` from it (toolchain per `arch/fad/diagrams/README.md`).
- Diagram changes: `cw_beacon` relabelled single-per-card (×1) + injection-point note; occupied-bin edges `≤9,600 → ≤9,830 occupied of 12,288-bin frame`; `filter_bank_synth/analysis` and `band_doppler_dl/ul` dual-valued ASIC/FPGA; subtitle carries the ×2-replication-scope + ASIC-nominal banner.

---

## B5 — OPTIONAL housekeeping: change-log row  ·  `arch/fad/FAD.md`

```
| 0.7.3 | 2026-05-28 | Marco Pausini | **Chapter-2.2 review fixes (pre-G1).** CW beacon insertion-point parent-doc conflict resolved via source hierarchy (ARCH SYS-GEN-07 Open/OI-B19 delegates injection point to SFU-001 §6.8) — captured as source erratum; FAD keeps post-synthesis/pre-Band-Gain/pre-Band-Doppler placement. `cw_beacon` clarified as single per-card block (not per-sub-band) in §2.2 prose, Figure 2.2 caption, and `subband_processing.drawio`. §2.2 RF BW reworded to ~1,638.4 MHz aggregate usable (~819.2 MHz/sub-band); occupied bin count corrected 9,600 → ~9,830 (80% of 12,288 frame) with [INFERRED] derivation. Figure 2.2 / diagram numeric labels flagged ASIC-nominal (FPGA ×1.024; bin counts invariant); `filter_bank_*` and `band_doppler_*` dual-valued. `subband_processing.drawio` updated + PNG re-rendered. |
```

---

## Post-apply check

```sh
grep -n "single per-card DL-only block" arch/fad/FAD.md          # B1
grep -n "OI-B19, which delegates" arch/fad/FAD.md                # B1 erratum
grep -c "9,600\|19,200 RF bins" arch/fad/FAD.md                  # expect 0 (B2/B2-alt)
grep -n "up to ~9,830 are occupied" arch/fad/FAD.md              # B2
grep -c "produce the 1,600 MHz" arch/fad/FAD.md                  # expect 0
grep -n "single per-card block (not replicated)" arch/fad/FAD.md # B3 caption
# diagram
grep -c "9,600\|1.024 GHz native\|±512 Hz" arch/fad/diagrams/subband_processing.drawio  # expect 0
```

Render §2.2 + the new PNG and eyeball: cw_beacon shown as one per-card block, beacon still post-synthesis/pre-gain, bin labels 9,830.
