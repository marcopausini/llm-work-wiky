# Change-set — FAD batch 07 (Chapter 5) — REBASED onto post-batch-05 state

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Pre-flight:** batches 01–06 applied (incl. batch 05 B1/B2/B3). §5.4 is in post-05 state. **Use this file instead of the original batch 07** — Group A FINDs are rebased onto post-05 text. Groups B and C are unchanged (untouched by 05/06).

Status note: batch 05 already resolved the reviewer's two §5 blockers (ADR-0010 collision via B1/B3; `mgmt_if` HP path via B2) and the sizes. Group A here is therefore only your **simplification on top** — folding the capture/playback bulk-data intent into the PL-DDR4 row and tightening the PS-DDR4 wording. If you're happy with the post-05 §5.4 as-is, A1/A2 are optional polish; A3 (QSPI) and A4 (capture/event_log) are the substantive remaining fixes.

---

## GROUP A — §5.4 simplification (rebased)  ·  `arch/fad/FAD.md`

### A1 — PL DDR4 row: fold in capture/playback bulk-data intent  ·  [APPROVED]

**FIND (post-05):**
```
| **PL DDR4** (10 GB, 320b @ 300 MHz PL clock / 2666 MT/s, ~76.7 Gbps usable; HW Guide §7.2) | **Reserved; activation deferred.** No MIG IP in the v0.7 bitstream; no PL DDR4 XDC; board PL DDR4 bring-up is off the critical path. | Candidate consumers: `capture` extended variant, `event_log` extended-persistence variant. Activation gated by OI-S08 (FAD-OQ-12); opens a dedicated PL-DDR4-use ADR at that point (number TBD — **not** ADR-0010, which is the clock-tree decision). | Deferred until activation. |
```
**REPLACE:**
```
| **PL DDR4** (10 GB, 320b @ 300 MHz PL clock / 2666 MT/s, ~76.7 Gbps usable; HW Guide §7.2) | **Reserved; deferred.** No MIG IP in the v0.7 bitstream; no PL DDR4 XDC; board bring-up off the critical path. | **Proposed for capture / playback bulk data** (deferred) — PL-DMA → PL-DDR4 keeps high-rate IQ in the PL, off the PS interconnect / 1 GbE; also candidate for `event_log` spill. Activation gated by OI-S08 (FAD-OQ-12); opens a dedicated PL-DDR4-use ADR (number TBD — **not** ADR-0010, the clock-tree decision). | Deferred until activation. |
```

### A2 — PS DDR4 row: tighten wording  ·  [APPROVED]

Batch 05 B2 already removed the `mgmt_if` path; this only adds the explicit "bulk data targets PL DDR4" pointer and a cleaner bandwidth cell.

**FIND (post-05):**
```
| **PS DDR4** (16 GB @ 2100 MT/s, 72-bit = 64 data + 8 ECC; HW Guide §7.2) | Out of SFU PL allocation scope. | PS-owned (Linux, mgmt protocol stack, TLE/SGP4 propagation per FAD-ARCH-01). No committed SFU PL→PS bulk path: `[TBD: capture/playback bulk-data path — leaning PL-DMA → PL-DDR4 (keep high-rate IQ in PL, off the PS interconnect / 1 GbE); a PS-DDR4-via-HP path is not preferred. Resolve at capture/playback MS authoring. owner: Marco]`. | Capture-download bandwidth budget deferred with the path decision (capture/playback MS scope). |
```
**REPLACE:**
```
| **PS DDR4** (16 GB @ 2100 MT/s, 72-bit = 64 data + 8 ECC; HW Guide §7.2) | Out of SFU PL allocation scope. | PS-owned (Linux, mgmt protocol stack, TLE/SGP4 propagation per FAD-ARCH-01). No committed SFU PL→PS bulk path — capture/playback bulk data targets PL DDR4 (deferred, see PL DDR4 row). | n/a — no SFU PL bulk path defined. |
```

*(If you prefer to keep the explicit `[TBD … owner: Marco]` marker that batch 05 B2 placed, skip A2 — the marker already states the deferral. A2 only trades the verbose marker for the shorter pointer.)*

### A3 — QSPI row: ownership precision  ·  [APPROVED]  (unchanged from original batch 07; not touched by 05/06)

**FIND:**
```
| **QSPI** (2 Gbit / 256 MB; HW Guide §7.3) | In scope, single owner. | [`nv_cfg`](../modules/nv_cfg.md) only — non-volatile config restore per SFU-001 §17.2. Shares the physical device with the boot image (BOOT.BIN / image.ub / boot.scr per HW Guide §11.2.1); partition coordination is `nv_cfg` MS scope. | Latency-tolerant; bandwidth budget = `nv_cfg` MS scope. |
```
**REPLACE:**
```
| **QSPI** (2 Gbit / 256 MB; HW Guide §7.3) | In scope; single PL owner of the `nv_cfg` partition. | [`nv_cfg`](../modules/nv_cfg.md) owns its config-restore partition only (SFU-001 §17.2). The PS boot image (BOOT.BIN / image.ub / boot.scr per HW Guide §11.2.1) shares the physical device and is **PS-owned**; partition layout and PS↔`nv_cfg` coordination authority are `nv_cfg` MS scope. | Latency-tolerant; bandwidth budget = `nv_cfg` MS scope. |
```

### A4 — §5.4 closing line  ·  [APPROVED]

Batch 05 B3 already fixed the ADR-0010 ref here; this adds the "authorable only once OI-S08 fixes the bounded contract" tightening.

**FIND (post-05):**
```
`capture` and `event_log` MSs are authored under §5.3 type allocation today. The DDR-backed extended variant carries `[STUB: OI-S08]` and is gated by the PL-DDR4 activation decision (FAD-OQ-12; dedicated ADR to be assigned — not ADR-0010); the URAM-bounded variant is authorable now.
```
**REPLACE:**
```
`capture` and `event_log` MSs are authored under §5.3 type allocation today. The DDR-backed extended variant carries `[STUB: OI-S08]` and is gated by the PL-DDR4 activation decision (FAD-OQ-12; dedicated ADR to be assigned — not ADR-0010). The URAM-bounded variant becomes authorable only once OI-S08 fixes its bounded minimum contract (capture points, depth, trigger modes, retention, download interface); until then both blocks remain `not_design_ready` (§6).
```

---

## GROUP B — obg_link FIFO resource type  ·  [APPROVED]  (unchanged — not touched by 05/06)

### B1 — §5.3 obg_link row

**FIND:**
```
| [`obg_link`](../modules/obg_link.md) | Aurora ↔ DSP CDC FIFOs (4 lanes × 2 directions); Aurora user-clock side FIFOs internal to vendor IP | LUTRAM | -- | Mechanism in [§4.5.3 row 1–4](#453-cdc-inventory); depth uArch. Aurora 64B/66B IP internal memory not separately budgeted (vendor IP). |
```
**REPLACE:**
```
| [`obg_link`](../modules/obg_link.md) | Aurora ↔ DSP CDC FIFOs (4 lanes × 2 directions); Aurora user-clock side FIFOs internal to vendor IP | `[TBD: FIFO resource type (LUTRAM / BRAM / URAM) from minimum depth/width analysis — uArch, owner: RTL]` | -- | Mechanism in [§4.5.3 row 1–4](#453-cdc-inventory); depth and type are uArch (LUTRAM only if the proven minimum depth ≤ 64 per §5.2). Aurora 64B/66B IP internal memory not separately budgeted (vendor IP). |
```

---

## GROUP C — recommended minor fixes  ·  [RECOMMENDED]  (unchanged — not touched by 05/06)

### C1 — §5.3 playback magnitude → TBD (OI-S05)

**FIND:**
```
| [`playback`](../modules/playback.md) | Pre-recorded LTE IQ frame buffer, pre-`bins_sel_dl` | URAM | O(few URAM) | Minimum 10 ms at largest configured BW (SFU-001 §8.1, SFU-DBG-02). Exact depth pending OI-S05 — see FAD-OQ-13. |
```
**REPLACE:**
```
| [`playback`](../modules/playback.md) | Pre-recorded LTE IQ frame buffer, pre-`bins_sel_dl` | URAM (estimate) | `[TBD: depth/resource from OI-S05 — sample format + largest stored waveform, owner: Architect]` | Minimum 10 ms at largest configured BW (SFU-001 §8.1, SFU-DBG-02); see FAD-OQ-13. |
```

### C2 — §5.3 capture magnitude → provisional (OI-S08)

**FIND:**
```
| [`capture`](../modules/capture.md) | IQ capture ring buffer at configurable taps (DL + UL) | URAM (bounded variant) **+** PL DDR4 (extended variant, deferred) | URAM O(several URAM); PL DDR4 O(GB) | `not_design_ready` per [§6.2.l](#62-notes-on-the-inventory). PL DDR4 activation deferred per §5.4 and FAD-OQ-12. MS authored with two architectural variants. |
```
**REPLACE:**
```
| [`capture`](../modules/capture.md) | IQ capture ring buffer at configurable taps (DL + UL) | URAM (bounded variant) **+** PL DDR4 (extended variant, deferred) | `[provisional, OI-S08]` URAM O(several URAM); PL DDR4 O(GB) | `not_design_ready` per [§6.2.l](#62-notes-on-the-inventory). PL DDR4 activation deferred per §5.4 and FAD-OQ-12. MS authored with two architectural variants. |
```

### C3 — §5.1 `[P5]` citation pointer (medium)

**FIND:**
```
XCZU43DR-2FFVE1156E on-chip memory primitives ([P5]; cross-checked against BittWare reference design post-implementation utilisation, HW Guide Table 7):
```
**REPLACE:**
```
XCZU43DR-2FFVE1156E on-chip memory primitives (device data sheet [P5], §14.2 references; cross-checked against BittWare reference design post-implementation utilisation, HW Guide Table 7):
```

---

## GROUP D — OPTIONAL change-log row

```
| 0.7.7 | 2026-05-28 | Marco Pausini | **Chapter-5 review fixes (pre-G1).** §5.4 PL DDR4 row folded the capture/playback bulk-data intent (PL-DMA→PL-DDR4, off PS) into the deferred row; PS DDR4 wording tightened (no SFU PL→PS bulk path; bulk data targets PL DDR4). QSPI ownership narrowed to the `nv_cfg` partition (PS boot image PS-owned; partition authority = nv_cfg MS). capture/event_log URAM-bounded variant gated on the OI-S08 bounded contract (remain not_design_ready). obg_link Aurora↔DSP FIFO resource type deferred to uArch (was unbounded LUTRAM vs §5.2). playback magnitude + capture allocation marked provisional (OI-S05 / OI-S08); §5.1 [P5] pointed to §14.2. (ADR-0010 collision + mgmt_if HP path already resolved in batch 05 B1–B3.) |
```

---

## Post-apply check

```sh
grep -n "Proposed for capture / playback bulk data" arch/fad/FAD.md   # A1
grep -n "bulk data targets PL DDR4 (deferred, see PL DDR4 row)" arch/fad/FAD.md  # A2
grep -n "single PL owner of the \`nv_cfg\` partition" arch/fad/FAD.md # A3
grep -n "becomes authorable only once OI-S08" arch/fad/FAD.md         # A4
grep -n "FIFO resource type (LUTRAM / BRAM / URAM)" arch/fad/FAD.md   # B1
grep -c "ADR-0010" arch/fad/FAD.md   # only legit §4 clock-tree refs remain
grep -c "mgmt_if\|tle_compute" arch/fad/FAD.md   # expect 0 (confirms 05+06 applied cleanly)
```
