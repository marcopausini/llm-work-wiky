# bpms-sfu-architecture — canonical state

**Last updated:** 2026-05-28
**Mirrored to Claude Project:** BPMS SFU Architecture

## Current phase

**G1 (Orient) review in progress.** FAD v0.8 (§1–§6 substantively complete; §7–§12 skeleton; §3 collapsed into §2) delivered for review on 2026-05-28 after a 9-batch pre-G1 cleanup pass.

**Active work:** adjudicate G1 reviewer feedback; commit FAD v0.8; accept the four newly-proposed ADRs gating first MSs (0004, 0012, 0013, 0014); apply diagram manual edits (`top_datapath` bin/BW numbers; `board_clock_scheme` MMCM fields).

**Next milestone:** G1 sign-off → baseline FAD → core ICDs (`streaming_bus`, `register_bus`, `obg_frame`) → first MS pilot.

## Artifacts (authoritative)

| Artifact | Location | Status |
|---|---|---|
| FAD | `arch/fad/FAD.md` | **v0.8** — §1–§6 complete; §5/§7–§12 skeleton |
| Spec layer | `arch/README.md` | final |
| Methodology | `docs/methodology/01..05-*.md` | complete (`03` stub — Gastón) |
| MS template | `arch/modules/_template.md` | initial; revision pending ADR-0005 confirmation |
| §2 diagrams | `arch/fad/diagrams/{top_datapath,subband_processing,infrastructure}.drawio` | current (label fixes pending manual edit) |
| §4 clock diagram | `arch/fad/diagrams/board_clock_scheme_v4.drawio` | label fix pending (MMCM fields) |
| repo-operator skill | `.claude/skills/repo-operator/` | stable |

## SFU/DCU boundary

**SFU owns:** OBG ingest, lane alignment, bins selection, filter-bank synth/analysis, band gain, band Doppler, RF interfacing, timing, management, debug.
**SFU does NOT own:** beam gain, beam Doppler, beam delay (DCU).
Authoritative table: FAD §1.2. Any boundary-crossing proposal requires an ADR with system-level citation.

## Module inventory — 20 MS files

- DL/UL identical contract → one parameterized MS, two RTL instances.
- `obg_link` — bidirectional, single MS (4 Aurora lanes full-duplex).
- `filter_bank_synth` / `filter_bank_analysis` — two separate MSs (different vendor IP).
- `tle_compute` → folded into `band_doppler` per **ADR-0013** (band_doppler owns 1PPS-aligned NCO load + intra-second interpolation; SGP4 stays in PS, bypasses `utc_apply_engine`).
- `mgmt_if` / `reg_bus` → retired as block nouns per **ADR-0012** (CSR fabric = AMD AXI SmartConnect IP + per-block AXI4-Lite slaves; no SFU-owned bridge or fabric block). The `register_bus` ICD survives as the per-block contract.
- `sfu_top` → no MS; top integration = FAD + constraints (FAD-ARCH-13).

Full inventory and instance count: FAD §6.1.

## Key structural commitments

- **Aurora line rate:** 19.6608 Gbps/lane (CPRI × 2; same GTY clock as DCU). 4 lanes per Q0 QSFP28 via 1:4 active SFP28 breakout. Parent docs (ARCH-001 / SFU-001) still state 19.2 — alignment pending.
- **Clocking (FAD §4):** four domains — `clk_rfdc_axis` (256 / 262.144 MHz), `clk_dsp` (341.333 / 349.525 MHz, MMCM `DIVCLK_DIVIDE=1, CLKFBOUT_MULT=4, CLKOUT0_DIVIDE=3` from `clk_rfdc_axis` — synchronous-derived, final per FAD-ARCH-08), `clk_aurora_user` (307.2 MHz, sat-invariant), `clk_mgmt` (100 MHz, sat-invariant).
- **Reset (FAD §4.4):** control-path-only synchronous resets (R1–R7); async-assert / sync-deassert lives only at the reset bridge per domain entry. R4: datapath unreset (preserves SRL/DSP-slice/CE packing).
- **CDC (FAD §4.5):** 13 data/control crossings + RDC-01..05 reset crossings; project-wide approved mechanism list in §4.5.1. ADR-0014 (pending) adds the MCP synchronised-enable load mechanism for the shadow-apply family (CDC-08).
- **Sat-switch:** planned re-commissioning event, no live switching. Aurora preserved (Si5394 OUT0/OUT1 multisynth N held); `clk_rfdc_axis` and `clk_dsp` re-lock at new frequencies. Sat-rate-dependent CSRs: `band_doppler_dl/ul` (incl. intra-second interp step), `cw_beacon`, possibly `band_rms_det`.
- **PS scope boundary:** PL RTL only in this repo. PS owns Ethernet GEM, mgmt protocol stack, TLE/SGP4 propagation, and clock-device writes (Si5394 over PS I²C; LMK04821 + LMX2594 via PS → AXI4-Lite → PL SPI bridge inside `clock_ctrl`).
- **MS / uArch split (ADR-0005):** architect delivers MS at `arch/modules/<m>.md` (black-box external contract; G4a `design_ready`); designer delivers uArch at `rtl/<m>/uarch.md` (FSMs/pipeline/FIFO sizing; G4b).
- **Sub-band:** two parallel chains per direction (~819.2 MHz usable each, ~9,830 occupied bins per sub-band). 48 beam slots per sub-band, 96 per SFU.

## ADR pipeline

| ADR | Topic | Status |
|---|---|---|
| 0001 | 5-doc Markdown framework | accepted |
| 0004 | `utc_apply_engine` (D+C miss handling) | proposed — gates first UTC-scheduled-param MS |
| 0005 | MS / uArch split | proposed — pending Gastón |
| 0006 | Filter-bank engine selection | proposed — gated on OQ-§4-9 (Shlomi) |
| 0007 | LMX2594 reference-frequency options | proposed v0.4 (A.1 / A.2 / B forwarded) |
| 0008 | LMK04821 reference-frequency options | pending — gated on FAD-OQ-11 (HW Lead) |
| 0009 | ASIC/FPGA sat sample-rate switching | accepted |
| 0010 | RFSoC clock-tree final + sat-switch sequencing | accepted |
| 0011 | BPMS 0.5 vs 1.0 clock-tree baseline | accepted |
| 0012 | Management CSR fabric = AXI SmartConnect | proposed — gates first S_AXI-Lite slave MS |
| 0013 | Fold `tle_compute` into `band_doppler` | proposed |
| 0014 | Add MCP synchronised-enable load to §4.5.1 mechanisms | proposed — gates CDC-08 sign-off |

ADR-0002 (clock architecture) and ADR-0003 (PS-PL partitioning) never crystallised — facts captured in FAD §4 and FAD-ARCH-01.

## Open carry-forwards

| ID / item | Resolves at |
|---|---|
| FAD-OQ-01 — `bins_sel` shared DL/UL or independent | Shlomi, before `bins_sel` MS |
| FAD-OQ-03 — UTC apply miss recovery | ADR-0004 acceptance |
| FAD-OQ-11 — LMK04821 vs LMK04828 | HW Lead — gates ADR-0008 |
| FAD-OQ-12 — PL DDR4 activation; PL-DMA→PL-DDR4 leaning for capture/playback bulk data | OI-S08; dedicated ADR |
| FAD-OQ-13 — `playback` storage depth | OI-S05 |
| FAD-OQ-14 — `event_log` persistence mechanism | OI-S08 |
| OQ-§4-10 — SFU-001 §11.1 Solution A vs B | Shlomi |
| OQ-§4-11 — GPS holdover budget | Sys Arch |
| OI-S10 — `spur_det` / `spectrum_monitor` merge | system |
| `register_bus` vs `mgmt_regmap` ICD redundancy | ICD pass |
| §4.5.1 mechanism list extension (MCP sync-enable) | ADR-0014 acceptance |
| Parent-doc realignment: SFU-001 §6.3 Option B erratum; 19.2→19.6608 line rate | Sys Arch (Shlomi) |
| SFU contract during antenna repoint (same-type handover) | Sys Arch (Shlomi) |

## Next concrete actions

1. Adjudicate G1 reviewer feedback; commit FAD v0.8.
2. NA
3. Accept ADRs 0004 / 0012 / 0013 / 0014 (gate first dependent MSs).
4. Author core ICDs: `streaming_bus`, `register_bus`, `obg_frame`. Resolve `register_bus` vs `mgmt_regmap` in the same pass.
5. Pilot MS end-to-end (candidate: `band_gain` or `bins_sel`).
6. Trigger ADR-0003 at FAD §11 authoring.

## Load-bearing principles

- **Reset two-axis discipline.** Inside a clock domain, fabric resets are synchronous (R1). Async-assert / sync-deassert lives only at the reset bridge at each domain entry (R2). Datapath unreset (R4).
- **MultiSynth-per-output preserves Aurora at sat-switch.** Si5394 per-output dividers let OUT3 retune without glitching OUT0/OUT1 (MGT REFCLKs).
- **CDC table is enumeration-grade.** FAD §4.5.3 lists every crossing with mechanism + owner MS; per-signal protocol in MS; FIFO sizing in uArch. Bidirectionally consistent with `report_cdc` at sign-off.
- **Citation tiers.** Cited / `[INFERRED from §]` / `[ASSUMPTION: text, expiry trigger]`. Every claim falls into one.
- **Changelog rows = milestones.** Per-change rationale lives in git commits, not the doc changelog.

## Active references

- BPMS-1.0-ARCH-001 + BPMS-1.0-SFU-001 v03.01 Draft (HTML-primary) — Project KB
- RFX-8440A Hardware Reference Guide r2v2 — Project KB
- `arch/fad/FAD.md` v0.8; `arch/adr/0001..0014-*.md`; `arch/fad/diagrams/`
- `docs/methodology/01..05-*.md`
- `.claude/skills/repo-operator/`

## Milestone log

| Date | Milestone |
|---|---|
| **2026-05-28** | **FAD v0.8 delivered for G1 review.** 9-batch pre-G1 cleanup pass: ADRs 0012/0013/0014 proposed; `tle_compute` folded; `mgmt_if`/`reg_bus` retired; MMCM notation corrected; RDC inventory added; FAD-OQ-12/13/14 + FAD-ARCH-13 added; Aurora 19.6608 Gbps; CDC-08 reclassified as MCP sync-enable load; CDC-09 changed to toggle-based pulse synchroniser. |
| 2026-05-25 | ADR-0007 v0.4 baselined (LMX options A.1/A.2/B forwarded to ADR-0008/-0010). |
| 2026-05-21 | FAD §4 v0.6 baselined (clocking/reset/CDC + sat-switch); ADR-0010 / ADR-0011 accepted. |
| ≤ 2026-05-08 | FAD §1–§3 + initial §4 authoring; module inventory consolidated to 23 (later 20 after `tle_compute` fold); methodology + repo-operator skill delivered; 5-doc framework + MS/uArch split adopted. |