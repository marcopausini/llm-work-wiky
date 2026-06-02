---
id: 2026-05-01--bpms-sfu-architecture--fad-s1-s2-s6-revision
date: 2026-05-01
project: bpms-sfu-architecture
type: decision
status: ongoing
topics: [fad, adr-0005, module-inventory, datapath-diagram, rf-port-sel, sub-band-architecture, beam-capacity]
source_chat: claude-opus-4.6
---

# FAD §1/§2/§6 revision — ADR-0005 implementation + diagram rework

## Context

Revising FAD.md v0.1 sections §1, §2, §6 to comply with ADR-0005 (spec/uArch split, accepted 2026-04-29). Session spanned two chats: the first (compacted) locked all module-pair consolidation decisions, CDC FIFO disposition, sfu_top drop, and utc_apply_engine understanding; the second (this chat) produced the deliverables and iterated on diagram correctness.

## Key findings

- Module inventory reduced from 31 legacy rows to 23 MS files (~28 RTL instances) by consolidating DL/UL pairs with identical external contracts into single parameterized MSs and dropping CDC FIFOs + sfu_top.
- `obg_rx` + `obg_tx` → single bidirectional `obg_link` MS (same 4 Aurora lanes, full-duplex, one IP instance per lane).
- `bins_sel`, `band_gain`, `band_doppler`, `band_rms_det` each become one parameterized MS with two RTL instances (`_dl`, `_ul`).
- `filter_bank_synth` and `filter_bank_analysis` stay as two separate MSs — different vendor IP cores.
- `cdc_fifo_dl` and `cdc_fifo_ul` dropped: CDC is a system-level constraint in `obg_link` MS; FIFO sizing is uArch-scope per ADR-0005.
- `sfu_top` dropped: top-level integration is FAD scope (§2 + constraints); no separate MS. `rtl/sfu_top/uarch.md` ownership remains TBD (Marco).
- **`rf_port_sel` placement corrected:** moved from after `rfdc_wrap` to between `band_doppler` and `rfdc_wrap`. The RFX-8440A has 3 ADC/DAC pairs hardwired to physical SSMC connectors; `rf_port_sel` is a digital mux in PL fabric, not an analog switch. Confirmed against original SFU-001 Figure 5.1.
- **`band_rms_det` is a passive tap, not inline.** DL taps post-`band_gain_dl`; UL taps post-`band_doppler_ul` / pre-`band_gain_ul`. Dashed arrow, no output back into chain.
- **Dual sub-band paths shown explicitly in diagrams.** Two parallel FB + band_gain + band_doppler chains per direction, within Mermaid subgraph boxes. Still one MS per function (FAD-DEC-05); diagram shows the hardware parallelism.
- Beam capacity per SFU is BW-dependent: 48 beam slots per sub-band (16/lane × 3 lanes); multiple beams pack per 240-pt FFT slot at narrower BWs. RF bandwidth limit (1,600 MHz ÷ BW) is binding at ≥5 MHz.
- The "384/288" annotation in original Figure 5.1 is ambiguous — `[TBD: confirm with system architect (Shlomi)]`.
- UTC-scheduled vs Local Autonomous distinction clarified in infrastructure diagram: `utc_apply_engine` handles shadow→live commits; `tle_compute` writes directly to `band_doppler` live NCO register at 1PPS (bypasses engine entirely).
- Merged single-diagram attempt (DL+UL with shared blocks plotted once) failed in Mermaid — layout engine can't handle bidirectional flow through shared nodes cleanly. Final approach: two stacked diagrams (DL `flowchart LR`, UL `flowchart RL`) with shared blocks labelled `(shared instance)`.

## Decisions

- FAD-DEC-02 updated: two-view organization (combined DL+UL datapath + infrastructure), replacing three-view (DL/UL/infra).
- FAD-DEC-03 refined: per-pair MS convention (identical contract → one MS, two instances with `DIRECTION` param; different contract → two MSs).
- FAD-DEC-07 finalized: CDC ownership model — mechanism in `obg_link` MS, FIFO sizing in uArch.
- FAD-DEC-09 softened: `band_rms_det` is separate from `band_gain`, one MS, two instances.
- FAD-DEC-10 added: no `sfu_top` MS; top-level integration = FAD + constraints.
- `rf_port_sel` placement: between `band_doppler` and `rfdc_wrap` (digital mux, not analog switch).
- Part F (open questions) goes into FAD as new §7, not a separate document.
- Change log v0.2 entry drafted.

## Open questions

- `bins_sel` routing table: shared between DL and UL for the same AxC, or independent per direction? `[ASSUMPTION: shared]` — confirm with system architect.
- `384/288` beam annotation in original Figure 5.1: what does the slash mean exactly?
- `utc_apply_bus` ICD must be frozen before first UTC-scheduled-parameter MS (trigger: ADR-0004).
- `rtl/sfu_top/uarch.md` ownership: architect or RTL designer? TBD Marco.
- Two-copies-of-§2 maintenance burden (inline condensed + standalone authoritative): consider switching to thumbnail-only inline at next revision.

## Action items

- [ ] Render latest `datapath.md` diagrams (with sub-band subgraphs) and confirm Mermaid layout is acceptable
- [ ] Produce FAD §2.1 inline version once standalone diagram is approved
- [ ] Apply full deliverable to repo: Part A (§1), Part B (§2), Part C (§6), infrastructure.md, datapath.md, change log v0.2, new §7 open questions
- [ ] Delete legacy `arch/fad/diagrams/dl-datapath.md` and `ul-datapath.md`
- [ ] Confirm `384/288` annotation meaning with Shlomi
- [ ] Confirm `bins_sel` routing table sharing with system architect
- [ ] Trigger ADR-0004 (utc_apply_bus ICD) before authoring first UTC-scheduled-parameter MS

## Artefacts produced

- `fad-section-1-2-6-revision.md` — consolidated deliverable with Part A (§1), Part B (§2 inline), Part C (§6), Part D (delta table), Part E (diagrams), Part F (open questions), Part G (checklist). Note: §2 datapath diagrams in this file are superseded by the latest standalone version.
  → target: `projects/bpms-sfu-architecture/deliverables/fad-section-1-2-6-revision.md`
- `datapath.md` — latest standalone authoritative diagram (DL LR + UL RL, dual sub-band subgraphs, corrected rf_port_sel placement, quantitative annotations, beam capacity table). This is the current version pending render approval.
  → target: `arch/fad/diagrams/datapath.md`
- `infrastructure.md` — updated standalone infrastructure diagram (in the consolidated deliverable Part E.2, unchanged since first delivery).
  → target: `arch/fad/diagrams/infrastructure.md`

## Re-seed block

GOAL: Complete FAD §1/§2/§6 revision per ADR-0005 and commit to repo.
CONSTRAINTS:
- Target device: BittWare RFX-8440A (AMD XCZU43DR-2FFVE1156E, speed grade -2)
- ADR-0005 accepted 2026-04-29: MS = architect-owned external contract; uArch = designer-owned internal implementation
- Gastón's template confirmation (Task 8) is critical-path blocker for downstream MS authoring
PRIOR CONCLUSIONS:
- 23 MS files, ~28 RTL instances; module-pair consolidation rules locked
- rf_port_sel is a digital mux between band_doppler and rfdc_wrap, not after rfdc_wrap
- band_rms_det is a passive tap (not inline); DL taps post-gain, UL taps post-doppler/pre-gain
- Dual sub-band paths shown explicitly in diagrams via Mermaid subgraphs
- Merged single-diagram (shared blocks once) failed in Mermaid; two stacked diagrams is the final approach
- utc_apply_engine handles UTC-scheduled shadow→live; tle_compute writes directly to band_doppler NCO at 1PPS (Local Autonomous, bypasses engine)
CURRENT QUESTION: Render the latest datapath.md with sub-band subgraphs, confirm layout is acceptable, then produce FAD §2.1 inline version and commit full revision to repo.
