# Change-set — FAD batch 09 (final review pass: CDC / framework cleanup)

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Pre-flight:** batches 01–08 applied. Anchors are byte-exact against the updated FAD.
**Apply method:** `str_replace` exact find-replace. Preserve `→ × ↔ § ²` glyphs and backticks.

Groups: **A** CDC-08 mechanism · **B** CDC-09 mechanism · **C** §4.5.2 stale assumption · **D** PL SPI bridge → clock_ctrl · **E** pps_capture CSR · **F** §13.2 add OQ-12/13/14 · **G** §1.4.1 add FAD-ARCH-13 · **H** §5.2 stale count.

Non-hunk actions at the end: **(I)** diagram label replacements for you to apply manually; **(J)** ADR-pack inclusion list.

---

## GROUP A — CDC-08 → approved mechanism (blocker)  ·  `arch/fad/FAD.md`

§4.5.1 has no "shadow-active atomic" mechanism. The Band Doppler word is multi-bit, written ~1 s ahead and held quasi-static, then captured in `clk_dsp` on the CDC-11-synchronised 1PPS strobe — i.e. a **held-stable multi-bit datum captured by a synchronised enable** (multi-cycle-path pattern). That is *not* literally on the §4.5.1 list, so per §4.5.1's own rule it needs either a waiver entry or an ADR extension; the honest framing names the pattern and its quasi-static contract and flags the §4.5.1 gap rather than mislabelling it as an existing row.

### A1 — §4.5.3 CDC-08 row

**FIND:**
```
| CDC-08 | mgmt | dsp | Band Doppler data (PS-supplied, per-second) | Shadow-active atomic at 1PPS — protocol in `band_doppler` MS | `band_doppler` |
```
**REPLACE:**
```
| CDC-08 | mgmt | dsp | Band Doppler data (PS-supplied, per-second) | Multi-bit datum held quasi-static ≥ 1 s (PS writes the next word ~1 s ahead; not modified inside the apply window), captured in `clk_dsp` on the CDC-11-synchronised 1PPS enable — multi-cycle-path / data-held-stable contract, owner `band_doppler` MS. `[TBD: this held-stable + synchronised-enable pattern is not yet an explicit §4.5.1 row — add it to §4.5.1 or carry a `waivers/cdc/` entry per §4.5.1; owner: Architect]` | `band_doppler` |
```

---

## GROUP B — CDC-09 → pulse-safe mechanism (high)  ·  `arch/fad/FAD.md`

`clk_dsp` (341 MHz) → `clk_mgmt` (100 MHz): a 1-cycle dsp pulse (~2.9 ns) is narrower than the 10 ns `clk_mgmt` period and can be missed by a plain two-flop. §4.5.1 forbids stretched-pulse without quasi-static proof, so use a toggle-based pulse synchroniser or an event FIFO.

### B1 — §4.5.3 CDC-09 row

**FIND:**
```
| CDC-09 | dsp | mgmt | Event pulses (saturation, errors, etc.) | Two-flop + edge | `event_log` |
```
**REPLACE:**
```
| CDC-09 | dsp | mgmt | Event pulses (saturation, errors, etc.) | Toggle-based pulse synchroniser (`async_reg`) — dsp event toggles a level, two-flop synchronised into `clk_mgmt`, edge-detected there; or an async-FIFO event queue if events can burst faster than the mgmt drain. Plain pulse-stretch is insufficient (1-cycle dsp pulse < `clk_mgmt` period). | `event_log` |
```

---

## GROUP C — §4.5.2 stale ADR-0002 assumption (high)  ·  `arch/fad/FAD.md`

§4.0 / FAD-ARCH-08 made the synchronous-derived relationship final; this note still marks it `[ASSUMPTION … ADR-0002 + OQ-§4-3]` (which batch 06 missed). ADR-0002 does not exist.

### C1 — §4.5.2 sync-derived note

**FIND:**
```
- `sync-derived` is `[ASSUMPTION — see §4.0; expiry: ADR-0002 + OQ-§4-3]`. Constraint: `set_clock_groups -group {clk_dsp clk_rfdc_axis}` (NOT `-asynchronous`). Vivado analyses paths normally.
```
**REPLACE:**
```
- `sync-derived` is **final** per §4.0 / FAD-ARCH-08 (`clk_dsp` is MMCM-derived from `clk_rfdc_axis` with a fixed integer ratio). Constraint: `set_clock_groups -group {clk_dsp clk_rfdc_axis}` (NOT `-asynchronous`). Vivado analyses paths normally; the 4↔3 sample-count mismatch is absorbed by the synchronous elastic stage in `rfdc_wrap` (uArch), not a clock-domain FIFO.
```

---

## GROUP D — PL SPI bridge folded into clock_ctrl (high; Marco's call)  ·  `arch/fad/FAD.md`

No new §6 row: the PL-side SPI-master IP that programs LMK04821 + LMX2594 is within `clock_ctrl` scope.

### D1 — §4.2.4 SPI-bridge clause (clock_ctrl role paragraph, line ~497)

**FIND:**
```
**`clock_ctrl` role (MS scope).** PS-side software owns every clock-device write — Si5394 over PS I²C, LMK04821 + LMX2594 via the bittware driver through the PL SPI bridge. `clock_ctrl` is the PL-side companion that:
```
**REPLACE:**
```
**`clock_ctrl` role (MS scope).** PS-side software owns every clock-device write — Si5394 over PS I²C, LMK04821 + LMX2594 via the bittware driver through the PL SPI bridge. The **PL-side SPI-master IP** for the LMK/LMX devices is part of `clock_ctrl` (not a separate §6 module): its AXI4-Lite control surface and SPI pinout are `clock_ctrl` MS scope, so no standalone inventory row exists for it. `clock_ctrl` is the PL-side companion that:
```

### D2 — §6 clock_ctrl row note (row 19)

**FIND:**
```
| 19 | [`clock_ctrl`](../modules/clock_ctrl.md) | new | `mgmt` | SFU-001 §11.1, §11.2, §11.4 | draft | 1 clock-control block |
```
**REPLACE:**
```
| 19 | [`clock_ctrl`](../modules/clock_ctrl.md) | new | `mgmt` | SFU-001 §11.1, §11.2, §11.4 | draft | 1 clock-control block; includes the PL-side SPI-master IP for LMK04821/LMX2594 (PS drives it via AXI4-Lite + bittware driver) — see §4.2.4 |
```

---

## GROUP E — pps_capture sample-rate CSR (medium)  ·  `arch/fad/FAD.md`

`pps_capture`'s UTC counter is 1PPS-driven in `clk_mgmt` (1 Hz, rate-invariant). The "1 second in `clk_dsp` cycles" count belongs to `band_doppler` interpolation, not `pps_capture`.

### E1 — §4.6.2 list entry

**FIND:**
```
- `pps_capture` — UTC counter increment value (1 second in `clk_dsp` cycles changes).
```
**REPLACE:**
```
- `band_doppler_dl/ul` — intra-second interpolation step (1 second expressed in `clk_dsp` cycles changes with sample rate). *(The `pps_capture` UTC counter is 1PPS-driven in `clk_mgmt` at 1 Hz and is therefore rate-invariant — not listed here.)*
```

---

## GROUP F — §13.2 add FAD-OQ-12/13/14 (medium)  ·  `arch/fad/FAD.md`

§5 references these three; §13.2 stops at OQ-03. Append the rows (anchor on the OQ-03 row).

### F1 — append after FAD-OQ-03

**FIND:**
```
| FAD-OQ-03 | UTC-scheduled apply miss recovery: what does the SFU do when a UTC tick is missed for an armed shadow register? | Decided in [ADR-0004](../adr/0004-utc-apply-engine.md) (status: proposed): uniform **D (write-time guard) + C (hold-previous on in-flight miss)**; per-parameter late-apply (A) reserved as a future amendment. The A/B/C/D alternatives are analysed in ADR-0004 "Alternatives considered" (not §2.3). | Before authoring the first UTC-scheduled-parameter MS (`rf_port_sel` or `band_gain`) or freezing the `utc_apply_bus` ICD, whichever is first. Triggers ADR-0004 acceptance. | Sys Arch | open |
```
**REPLACE:**
```
| FAD-OQ-03 | UTC-scheduled apply miss recovery: what does the SFU do when a UTC tick is missed for an armed shadow register? | Decided in [ADR-0004](../adr/0004-utc-apply-engine.md) (status: proposed): uniform **D (write-time guard) + C (hold-previous on in-flight miss)**; per-parameter late-apply (A) reserved as a future amendment. The A/B/C/D alternatives are analysed in ADR-0004 "Alternatives considered" (not §2.3). | Before authoring the first UTC-scheduled-parameter MS (`rf_port_sel` or `band_gain`) or freezing the `utc_apply_bus` ICD, whichever is first. Triggers ADR-0004 acceptance. | Sys Arch | open |
| FAD-OQ-12 | PL DDR4 activation for `capture` / `event_log` bulk data (and `event_log` spill): is the PL-DMA → PL-DDR4 extended variant built, and on what data path? | `[ASSUMPTION: deferred]` — v0.7 baseline is URAM-bounded; PL DDR4 reserved, no MIG IP in the bitstream (§5.4). Leaning PL-DMA → PL-DDR4, off the PS. | OI-S08 resolution; opens a dedicated PL-DDR4-use ADR (number TBD — not ADR-0010). Before the `capture` extended-variant MS reaches `design_ready`. | Architect | open |
| FAD-OQ-13 | `playback` storage depth and resource: sample format + largest stored waveform fix the URAM/DDR depth. | `[TBD]` per §5.3 — URAM estimate only; 10 ms at largest configured BW known, exact depth not. | OI-S05 resolution. Before the `playback` MS reaches `design_ready`. | Architect | open |
| FAD-OQ-14 | `event_log` persistence mechanism: URAM-volatile vs QSPI-backed vs PL-DDR4 spill. | `[TBD]` per §5.3 — volatile URAM default. | OI-S08 resolution. Before the `event_log` MS reaches `design_ready`. | Architect | open |
```

---

## GROUP G — §1.4.1 add FAD-ARCH-13 (medium)  ·  `arch/fad/FAD.md`

FAD-METH-03 and §6.2.k cite FAD-ARCH-13 (top-level integration scope); the row is missing (the §15 mapping shows `DEC-10 → ARCH-13 + METH-03 (split)` but only METH-03 was added). Insert after FAD-ARCH-11.

### G1 — add FAD-ARCH-13 row

**FIND:**
```
| FAD-ARCH-11 | Parameter apply uses **two** mechanisms, selected per parameter and declared in each block's MS: (a) UTC-scheduled apply per [ADR-0004](../adr/0004-utc-apply-engine.md) for low-rate coordinated parameters; (b) 1PPS-aligned apply for operational Band Doppler — PS sw writes the next Doppler data ~1 s in advance; `band_doppler` (PL) latches and interpolates the NCO word at the 1PPS edge ([ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)). | [ADR-0004](../adr/0004-utc-apply-engine.md) (UTC path only); see §2.3 and §4 |
```
**REPLACE:**
```
| FAD-ARCH-11 | Parameter apply uses **two** mechanisms, selected per parameter and declared in each block's MS: (a) UTC-scheduled apply per [ADR-0004](../adr/0004-utc-apply-engine.md) for low-rate coordinated parameters; (b) 1PPS-aligned apply for operational Band Doppler — PS sw writes the next Doppler data ~1 s in advance; `band_doppler` (PL) latches and interpolates the NCO word at the 1PPS edge ([ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)). | [ADR-0004](../adr/0004-utc-apply-engine.md) (UTC path only); see §2.3 and §4 |
| FAD-ARCH-13 | Top-level integration scope is FAD-owned, not a module spec: top-level external ports are described in §2, and pin/package assignments and I/O standards live in `constraints/00_io.xdc`. (Architecture half of legacy FAD-DEC-10; the documentation half is FAD-METH-03.) | N/A — see §2 and FAD-METH-03 |
```

*(If FAD-ARCH-12 is also referenced anywhere — the §15 mapping shows `DEC-03 → ARCH-12 + METH-02` — confirm whether it needs a row too; I found no live `FAD-ARCH-12` reference in the body, so it is not added here.)*

---

## GROUP H — §5.2 stale count (low)  ·  `arch/fad/FAD.md`

### H1

**FIND:**
```
To keep memory inference consistent across 23 module specs and prevent designer-by-designer drift:
```
**REPLACE:**
```
To keep memory inference consistent across the §6 module inventory and prevent designer-by-designer drift:
```

---

## GROUP I — change-log row (optional)  ·  `arch/fad/FAD.md`

```
| 0.7.9 | 2026-05-28 | Marco Pausini | **Final review pass (pre-G1).** CDC-08 reworded to an approved multi-cycle-path / data-held-stable contract (1PPS-synchronised enable) with a §4.5.1-gap TBD; CDC-09 changed to a toggle-based pulse synchroniser / event FIFO (plain two-flop misses sub-`clk_mgmt`-period dsp pulses). §4.5.2 sync-derived note made final per §4.0/FAD-ARCH-08 (stale ADR-0002/OQ-§4-3 marker removed — missed in batch 06). PL SPI-master IP for LMK/LMX folded into `clock_ctrl` scope (no new §6 row) per §4.2.4 + row-19 note. `pps_capture` removed from §4.6.2 sample-rate CSR list (UTC counter is 1PPS-driven in clk_mgmt; clk_dsp-cycle count belongs to band_doppler). §13.2 gained FAD-OQ-12/13/14 (PL-DDR4 activation, playback depth, event_log persistence). §1.4.1 gained FAD-ARCH-13 (top-level integration scope; arch half of DEC-10). §5.2 "23 module specs" → "the §6 module inventory". Diagrams (top_datapath, board_clock_scheme) corrected manually. |
```

---

## Post-apply check

```sh
grep -c "Shadow-active atomic" arch/fad/FAD.md                 # expect 0
grep -n "multi-cycle-path / data-held-stable" arch/fad/FAD.md  # A1
grep -n "Toggle-based pulse synchroniser" arch/fad/FAD.md      # B1
grep -c "ADR-0002" arch/fad/FAD.md                             # expect 0
grep -n "PL-side SPI-master IP.*part of \`clock_ctrl\`" arch/fad/FAD.md  # D1
grep -c "pps_capture. — UTC counter increment value" arch/fad/FAD.md    # expect 0
grep -cE "FAD-OQ-1[234]" arch/fad/FAD.md                       # expect ≥3
grep -n "FAD-ARCH-13 | Top-level integration scope" arch/fad/FAD.md     # G1
grep -c "23 module specs" arch/fad/FAD.md                      # expect 0
```
