# Change-set — FAD batch 08 (§6 inventory renumber + FAD-OQ-03 + clock-domain shorthand)

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Pre-flight:** batches 01–07 applied. Group B's Total hunk targets **post-batch-04** text (batch 04 B11 set "Total: 22 … [TBD]"); the row hunks and Groups A/C target text untouched by prior batches.
**Apply method:** `str_replace` exact find-replace. Preserve `→ × § ²` glyphs and backticks.

Tags: **[APPROVED]** Marco confirmed · **[RECOMMENDED]** droppable.

Groups: **A** FAD-OQ-03 → ADR-0004 · **B** §6 renumber 1–20 · **C** clock-domain shorthand cleanup.

Withdrawn from the §6 review: note 6.2.l is already complete (lists all four `not_design_ready` blocks) — no fix needed.

---

## GROUP A — FAD-OQ-03 → ADR-0004  ·  [APPROVED]  ·  `arch/fad/FAD.md`

The miss-recovery is decided in ADR-0004 (proposed: D+C), not open `[TBD]`; and the A/B/C/D options live in ADR-0004, not §2.3.

### A1 — §13.2 FAD-OQ-03 "Current assumption / default" cell

**FIND:**
```
None — `[TBD]` per §2.3. Four candidate options listed in §2.3: **A** late-apply, **B** snap-to-next-1PPS, **C** hold-previous, **D** write-time guard. Composition allowed. Defensible default candidate: `D + C`.
```
**REPLACE:**
```
Decided in [ADR-0004](../adr/0004-utc-apply-engine.md) (status: proposed): uniform **D (write-time guard) + C (hold-previous on in-flight miss)**; per-parameter late-apply (A) reserved as a future amendment. The A/B/C/D alternatives are analysed in ADR-0004 "Alternatives considered" (not §2.3).
```

*(Resolution-trigger cell — "Triggers ADR-0004 acceptance" — and Status `open` stay as-is: ADR-0004 is proposed, not yet accepted.)*

---

## GROUP B — §6.1 renumber contiguous 1–20  ·  [APPROVED]  ·  `arch/fad/FAD.md`

Rows 1–15 unchanged. Former 17/19/20/21/22 → 16/17/18/19/20 (row 18 `tle_compute` already removed in batch 04; gaps at 16 and 18 closed). Row hunks anchor on the module name (unique).

### B1 — row 17 → 16 (`utc_apply_engine`)

**FIND:**
```
| 17 | [`utc_apply_engine`](../modules/utc_apply_engine.md) |
```
**REPLACE:**
```
| 16 | [`utc_apply_engine`](../modules/utc_apply_engine.md) |
```

### B2 — row 19 → 17 (`pps_capture`)

**FIND:**
```
| 19 | [`pps_capture`](../modules/pps_capture.md) |
```
**REPLACE:**
```
| 17 | [`pps_capture`](../modules/pps_capture.md) |
```

### B3 — row 20 → 18 (`event_log`)

**FIND:**
```
| 20 | [`event_log`](../modules/event_log.md) |
```
**REPLACE:**
```
| 18 | [`event_log`](../modules/event_log.md) |
```

### B4 — row 21 → 19 (`clock_ctrl`)

**FIND:**
```
| 21 | [`clock_ctrl`](../modules/clock_ctrl.md) |
```
**REPLACE:**
```
| 19 | [`clock_ctrl`](../modules/clock_ctrl.md) |
```

### B5 — row 22 → 20 (`nv_cfg`)

**FIND:**
```
| 22 | [`nv_cfg`](../modules/nv_cfg.md) |
```
**REPLACE:**
```
| 20 | [`nv_cfg`](../modules/nv_cfg.md) |
```

### B6 — Total line (POST-batch-04 anchor)

**FIND:**
```
**Total:** 22 module-spec rows. The corresponding top-level RTL may contain more than 22 instances because some module specs are reused for DL and UL instances. `[TBD: reconcile stated total against visible rows and the missing row-16 index — pre-existing §6.1 bookkeeping discrepancy, owner: Marco]`
```
**REPLACE:**
```
**Total:** 20 module-spec rows. The corresponding top-level RTL may contain more than 20 instances because some module specs are reused for DL and UL instances.
```

---

## GROUP C — clock-domain shorthand cleanup  ·  [RECOMMENDED]  ·  `arch/fad/FAD.md`

Three stale entries in the §6 preamble shorthand: `ps_axi` (collapsed into `mgmt`, FAD-ARCH-06; "management bridge" is retired `mgmt_if` language), `dsp_in`/`dsp_out` (contradict FAD-ARCH-07 single-clock filter-bank wrappers; no row uses them), and the `ADR-0002` provisional line (ADR-0002 does not exist). None of the three is used by any §6.1 row.

### C1 — provisional-line intro

**FIND:**
```
**Clock-domain shorthand:** final clock names and rates are owned by FAD §4. Until ADR-0002 / FAD §4 are baselined, the shorthand below is provisional.
```
**REPLACE:**
```
**Clock-domain shorthand:** final clock names and rates are owned by FAD §4; the shorthand below maps to those domains.
```

### C2 — remove `dsp_in`/`dsp_out` row

**FIND:**
```
| `dsp_in` / `dsp_out` | Filter-bank wrapper input/output processing domains. |
```
**REPLACE:** *(empty — remove the line including its trailing newline)*

### C3 — remove `ps_axi` row

**FIND:**
```
| `ps_axi` | PS-driven AXI clock domain for PS↔PL management bridge. |
```
**REPLACE:** *(empty — remove the line including its trailing newline)*

---

## GROUP D — OPTIONAL change-log row

```
| 0.7.8 | 2026-05-28 | Marco Pausini | **§6 renumber + FAD-OQ-03 + shorthand cleanup.** §6.1 inventory renumbered contiguous 1–20 (closed the 16/18 gaps left by the tle_compute fold; Total corrected 22→20, batch-04 reconcile TBD cleared). FAD-OQ-03 repointed to ADR-0004 (proposed D+C; alternatives are in ADR-0004, not §2.3 as the row wrongly claimed); status stays open pending ADR-0004 acceptance. Clock-domain shorthand cleaned: removed stale `ps_axi` (collapsed into `mgmt`, FAD-ARCH-06) and `dsp_in`/`dsp_out` (contradict single-clock filter-bank wrappers, FAD-ARCH-07), and dropped the non-existent ADR-0002 provisional reference. Note 6.2.l confirmed already complete (all four not_design_ready blocks listed). |
```

---

## Post-apply check

```sh
# renumber: contiguous 1..20, no gaps, no 21/22
grep -nE "^\| (1[6-9]|20) " arch/fad/FAD.md          # rows 16-20 present
grep -cE "^\| (21|22) " arch/fad/FAD.md              # expect 0
grep -n "Total:.. 20 module-spec rows" arch/fad/FAD.md
grep -c "missing row-16 index" arch/fad/FAD.md       # expect 0 (TBD cleared)
# FAD-OQ-03
grep -n "Decided in \[ADR-0004\]" arch/fad/FAD.md
grep -c "Four candidate options listed in §2.3" arch/fad/FAD.md   # expect 0
# shorthand
grep -c "ps_axi\|dsp_in\|ADR-0002" arch/fad/FAD.md   # expect 0 if Group C applied
```

Eyeball §6.1: 20 contiguous rows, `nv_cfg` is row 20, Total 20.
