---
id: 2026-04-21--bpms-sfu-architecture--micro-arch-and-workspace-baseline
date: 2026-04-21
project: bpms-sfu-architecture
type: reference
status: ongoing
topics: [sfu, micro-architecture, block-decomposition, cdc, fpga-partitioning, bpms]
source_chat: claude-sonnet-4-6
---

# SFU Micro-Architecture Baseline and Workspace Setup

## Context

Initial architecture review and decomposition from BPMS 1.0 System Architecture Document v2.4
and BPMS 1.0 SFU Architecture Document v1.6. Covered SFU functional boundary, full PL/PS block
decomposition, CDC map, document inconsistencies, and workspace/tooling decisions for the
spec-to-RTL pipeline.

## Key findings

### Document inconsistencies

- ARCH §7.3 section header says "Option B selected" but the decision body and all downstream
  references describe Option A (dual filter bank in DCU). Option A is the actual decision.
  Editorial error in the heading — do not propagate.
- CW Beacon injection point ambiguous. SFU §6.8 says "before Band Doppler and Band Gain"
  but §6.5 places Band Gain after FB Synthesis. Resolved reading:
  FB Synthesis → Beacon injection → Band Gain → Band Doppler.
- FB sub-band input expects 12,288 bins; each OBG lane carries 5,760 bins. Bins Selector
  aggregation rules from multiple OBG lanes into sub-band inputs are unspecified — IQ-1 blocker.
- FPGA sat filter bank BW: SFU doc states 1,048.576 MHz; system doc says TBD. Use SFU doc
  value as more specific and more recent.

### SFU functional boundary

- Owns: OBG ingest, lane alignment, bins selection/routing, filter-bank synthesis/analysis,
  band gain, band Doppler, CW beacon, RF interfacing, timing, management, debug.
- Does NOT own: beam gain, beam Doppler, beam delay — all DCU.
- Satellite-oriented: one sat type at a time; re-commissioning required to switch.
- FPGA sat ×1.024 scaling applies to: ADC/DAC rate (4,194.304 Msps), FB BW (1,048.576 MHz),
  Band Doppler range (±524.288 Hz). All derived timing parameters scale by the same factor.

### PL block decomposition (17 blocks)

aurora_rx_tx, dl_cdc, ul_cdc, lane_align, bins_sel_dl, bins_sel_ul, fb_synth (×2),
fb_analysis (×2), beacon_inj, band_gain (×4 — DL-A, DL-B, UL-A, UL-B), band_dop (×4),
duc_ddc, rf_port_sel, timing_core, param_engine, mgmt_regs, rms_detector, latency_meas,
playback, capture, spur_det.

### PS module decomposition (5 modules)

tle_doppler (SGP4 on ARM A53), mgmt_sw (1GbE agent), nv_config, fw_mgmt, event_log_mgr.

### CDC crossings identified

- 6 data-path crossings, 2 control-path crossings. Exact depths TBD pending clock budget.

### RTL-readiness split

- **Unblocked now:** param_engine, band_gain, band_dop, timing_core, rf_port_sel,
  latency_meas, lane_align, rms_detector, mgmt_regs skeleton.
- **Blocked:** aurora_rx_tx and bins_sel (need OBG frame ICD — IQ-1), fb_synth/fb_analysis
  (need filter bank IP — IQ-2), capture/playback (need OI-S05/OI-S08 resolved).

### Artifacts generated

- `BPMS_1_0_sfu_architecture_review.md` — engineering review: system summary, SFU scope,
  signal chain, requirements, ambiguities, gaps, recommendations.
- `BPMS_1_0_sfu_micro_architecture.md` — block-level micro-architecture: PL/PS split,
  interfaces, clock domains, CDC map, control taxonomy, debug/telemetry, resource estimate,
  open implementation questions.

## Decisions

- **D1** Use Obsidian + Claude Code for spec/architecture work (not Claude Chat). Rationale:
  project will reach 30–50 interlinked artifacts; chat context is ephemeral and token-limited.
- **D2** Workspace: `bpms-sfu/` parent with `code/` and `vault/` as separate Git repos.
  Independent commit cadences; Claude Code at workspace root sees both trees.
- **D3** Naming: hyphens for folder/repo names; short local names (`code/`, `vault/`);
  descriptive remotes (`bpms-sfu-rtl`, `bpms-sfu-vault`).
- **D4** Source doc conversion: prefer HTML export over Word; Pandoc for conversion.
  HTML → MD is nearly lossless; Word → MD leaves artifacts.
- **D5** TLE→Doppler compute allocated to PS (ARM A53). Rationale: SGP4 needs
  double-precision float; NCO apply stays in PL.
- **D6** `CLAUDE.md` at workspace root outside both repos, optionally tracked in vault repo.

## Open questions

### Implementation

- **IQ-1 [Critical]** OBG Aurora frame format: header, bin ordering, metadata, CRC.
  No ICD exists — must be authored with DCU team.
- **IQ-2 [Critical]** Filter bank IP source, delivery date, integration guide (ref [7]).
- **IQ-3 [Critical]** Management register map / protocol (OI-S07).
- **IQ-4 [Critical]** Single or dual FPGA bitstream for ASIC/FPGA satellite modes.
- **IQ-5 [High]** Bins Selector mapping: full crossbar or constrained routing?
- **IQ-6 [High]** CW Beacon: one NCO per SFU or one per sub-band?
- **IQ-7 [High]** Sub-band A+B combining before DUC: explicit combiner block or
  RFSoC tile multi-band mode?
- **IQ-8 [High]** IQ word width and bit-growth budget through full pipeline.
- **IQ-9 [High]** Playback data format: bin-domain confirmed? Memory depth? (OI-S05)
- **IQ-10 [High]** Capture memory depth and DMA architecture (OI-S08).
- **IQ-11 [High]** Inter-SFU phase skew budget → sets lane_align FIFO depth (OI-S02).
- **IQ-12 [Medium]** CDC FIFO depth for secondary clock mode (10 MHz CI path).
- **IQ-13 [Medium]** Spur detection vs spectrum monitor resource sharing (OI-S10).
- **IQ-14 [Medium]** Loopback mode feasibility (OI-S09).

### Workspace setup

- **WS-1** Starting on remote server via VS Code SSH?
- **WS-2** Workspace root path (e.g. `~/projects/bpms-sfu/`)?
- **WS-3** Git remote: GitHub, GitLab, or internal?

## Action items

- [ ] Set up workspace: `bpms-sfu/` with `code/` and `vault/` sub-repos (A1)
- [ ] Convert source docs (HTML preferred) to Markdown via Pandoc (A2)
- [ ] Populate vault skeleton using agreed folder structure (A3)
- [ ] Set up Git remotes, clone to laptop, configure Obsidian (A4–A5)
- [ ] Configure Claude Code at workspace root; draft `CLAUDE.md` (A6–A7)
- [ ] Author OBG frame ICD with DCU team — Priority 1 blocker (A8)
- [ ] Obtain filter bank IP from DSP Lead (ref [7]) — Priority 1 blocker (A9)
- [ ] Start RTL block specs for unblocked blocks: param_engine, band_gain,
      band_dop, timing_core (A10) — after workspace is up

## Re-seed block

```
GOAL: Produce RTL-ready block specs for the BPMS 1.0 SFU starting with unblocked PL blocks.
CONSTRAINTS:
- Target device: AMD Zynq UltraScale+ RFSoC XCZU43DR-2FFVE1156E (BittWare RFX-8440A)
- Band-level processing only — no per-beam corrections (those are DCU)
- Satellite-oriented: one sat type per SFU at a time; FPGA sat uses ×1.024 scaling throughout
- Source of truth: BPMS 1.0 System Architecture Document v2.4 + SFU Architecture Document v1.6
PRIOR CONCLUSIONS:
- Option A (dual filter bank in DCU) is the actual FPGA sat grid alignment decision — ARCH §7.3 heading is wrong
- CW Beacon injection order: FB Synthesis → Beacon → Band Gain → Band Doppler
- 17 PL blocks + 5 PS modules identified; param_engine/band_gain/band_dop/timing_core/
  rf_port_sel/latency_meas/lane_align/rms_detector unblocked for spec now
- aurora_rx_tx and bins_sel blocked on OBG frame ICD (IQ-1); fb_synth/fb_analysis blocked on IP (IQ-2)
- TLE→Doppler computation allocated to PS ARM A53; NCO apply in PL
- Workspace: bpms-sfu/ parent, code/ and vault/ sub-repos, Claude Code at workspace root
CURRENT QUESTION: Which unblocked block gets the first RTL-ready block spec, and does the
  sfu_block_spec_template.md exist in the vault yet?
```
