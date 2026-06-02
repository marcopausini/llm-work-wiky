---
id: 2026-05-07--bpms-sfu-architecture--fad-4-clocking-reset-policy
date: 2026-05-07
project: bpms-sfu-architecture
type: decision
status: resolved
topics: [fad, clocking, reset, ug949, cdc, mmcm, rfdc, sample-rate-switching, control-path-reset, filter-bank]
source_chat: claude-opus-4.7
---

# FAD §4 — Clocking and Reset Architecture authored; control-path-only reset adopted

## Context

Authored FAD §4 (Clocking and Reset Architecture) for the SFU FPGA, integrating ADR-0006 (filter-bank engine selection — FFT4096 pair) and ADR-0007 (single-bitstream sample-rate switching) with AMD UG949 *Clocking Guidelines* and *Resets* recommendations. Replaces the prior v0.3 skeleton with a complete §4 ready for FAD v0.5 integration.

## Key findings

- ADR-0006 (FFT4096 pair) collapses the filter-bank IP into a single-clock interface at the wrapper boundary — `clk_fb_sample` domain eliminated, no internal CDC contributed by the FB IP.
- ADR-0007 topology (DSP MMCM derived from `clk_rfdc_axis` with fixed M/D ratio) makes `clk_dsp` ↔ `clk_rfdc_axis` **synchronous-derived 1:3 by construction** — no async FIFO at the `rfdc_wrap` boundary; pipeline registers under normal Vivado timing analysis.
- Sample-rate ×1.024 scaling (ASIC ↔ FPGA-sat) propagates structurally through the MMCM cascade — single bitstream, no DRP, no second MMCM, no BUFGMUX.
- Aurora SERDES domain (`clk_aurora_user`) is invariant across sample-rate switch; Aurora link preserved across the reconfiguration window (ADR-0007).
- CDC inventory reduced from 15 (prior draft) to **13 entries**: −2 from FB single-clock, −2 from RFDC↔DSP synchronous; +0 net additions.
- Control-path-only reset policy adopted as a **hard architectural rule** (rules R1–R6 in §4.4.1), grounded in UG949 §"Resets".
- DSP datapath registers explicitly NOT reset: filter-bank pipeline, NCO accumulators, multiplier outputs, RMS accumulators, AXI-Stream pipeline registers. First valid sample flushes any X-state; configuration valid bit gates output until loaded.
- Reset inventory reduced from 5 (prior draft) to **4 resets**: `rst_mgmt_n`, `rst_dsp_ctrl_n`, `rst_aurora_n`, `rst_rfdc_n`.
- `ps_axi` domain collapses into `mgmt` (pl_clk0 used for both PS-AXI master and PL register-bus) — eliminates one CDC boundary.
- Sample-rate-switch sequence asserts `rst_dsp_ctrl_n` and `rst_rfdc_n` across the reconfiguration window; `rst_mgmt_n` and `rst_aurora_n` remain deasserted (mgmt drives the switch; Aurora link preserved).
- UTC counter semantics: reset only by `rst_mgmt_n` at PS boot; never by sample-rate-switch or mode change. Loaded once at commissioning, incremented by 1PPS thereafter.
- Vivado `set_clock_groups -asynchronous` covers `clk_dsp`/`clk_aurora_user`/`clk_mgmt`; `clk_dsp`/`clk_rfdc_axis` deliberately NOT in async-groups (synchronous-derived).
- Reset bridge (async assert / sync deassert) with `async_reg = "true"` is the project-wide synchroniser pattern per UG949.

## Decisions

- **FAD-DEC-11** confirmed: collapse `clk_ps_axi` into `clk_mgmt` (pl_clk0 drives both). Recommended default; pending `mgmt_if` MS confirmation.
- **FAD-DEC-14** confirmed: `filter_bank_synth` and `filter_bank_analysis` are single-clock wrappers at `clk_dsp`; no internal CDC. Per ADR-0006.
- **FAD-DEC-15** confirmed: single bitstream supports both ASIC-sat and FPGA-sat via RFdc `DynamicPLLConfig` + MMCM cascade. Per ADR-0007.
- **FAD-DEC-16** confirmed: control-path-only reset policy is a **hard architectural rule**. Every MS conforms; deviations require explicit justification recorded in MS §10 and reviewed at G4a.
- **FAD-DEC-17** confirmed: `clk_dsp` ↔ `clk_rfdc_axis` synchronous-derived 1:3 via MMCM. Consequence of ADR-0007.
- UTC counter reset semantics: reset only by `rst_mgmt_n` at PS boot. Confirmed.
- §6 clock-domain shorthand updated: `aurora`/`dsp`/`rfdc`/`mgmt` (4 names). `dsp_in`/`dsp_out` retired (single-clock FB IP); `ps_axi` retired (FAD-DEC-11).

## Open questions

- **OQ-§4-1** Pin `clk_mgmt` frequency at FAD §11 baseline. Recommend `pl_clk0 = 100 MHz`. Owner: Architect. Expiry: FAD §11 authoring.
- **OQ-§4-2** Pin `clk_aurora_user` frequency once Aurora IP user-interface width is selected (64-bit ≈ 290.9 MHz, 128-bit ≈ 145.5 MHz). Owner: RTL Designer. Expiry: first `obg_link` MS draft (G4a).
- **OQ-§4-3** Confirm MMCM VCO range valid at both `clk_rfdc_axis` values × chosen M. Owner: RTL Designer. Expiry: first OOC synthesis run.
- **OQ-§4-4** Confirm BittWare RFX-8440A LMK/LMX supports 1.024-scalable reference output to RFdc tiles + separate fixed-rate reference for Aurora GT. Owner: HW Lead. Expiry: before bring-up; **gates ADR-0007 acceptance**.
- **OQ-§4-5** Decide MTS (Multi-Tile Sync) usage — affects switch-sequence step S5. Owner: System Architect. Expiry: before mgmt-plane state-machine spec.
- **OQ-§4-6** Confirm `rst_mgmt_n` / `rst_dsp_ctrl_n` fanout strategy (whether explicit pipelining is required). Owner: RTL Designer. Expiry: first top-level synthesis.
- **OQ-§4-9** Reconcile SFU-001 v1.6 §6.4/§7.4 wording (FLEXFFT240 vs FFT4096). Owner: System Architect (Shlomi). Expiry: **gates ADR-0006 acceptance**.

## Action items

- [ ] Integrate `fad_section4_v05.md` into `arch/fad/FAD.md` replacing current §4 skeleton (lines 337–367); bump FAD version 0.4 → 0.5.
- [ ] Add FAD-DEC-11, 14, 15, 16, 17 to FAD §1.4 decisions list with ADR pointers.
- [ ] Update FAD §6 inventory: confirm `filter_bank_synth` and `filter_bank_analysis` classified as `wrap`; update notes to reference CPBF FFT4096 block names per ADR-0006.
- [ ] Update FAD §6 clock-domain shorthand table per §4.1.1 (4 domains; retire `dsp_in`/`dsp_out` and `ps_axi`).
- [ ] Update `state.md`: FAD §4 complete; FAD-DEC list extended (5 new); CDC inventory at 13 entries; reset philosophy confirmed; sample-rate-switch sequencing pointer in place.
- [ ] Reconcile SFU-001 §6.4/§7.4 wording with Shlomi (OQ-§4-9; gates ADR-0006 acceptance).
- [ ] Confirm RFX-8440A LMK/LMX 1.024-scalable capability with HW Lead (OQ-§4-4; gates ADR-0007 acceptance).
- [ ] Trigger ADR-0002 (clock architecture) authoring to formally capture FAD-DEC-11 and FAD-DEC-16.
- [ ] Next chat: start core ICDs (streaming_bus, register_bus, obg_frame). Trigger ADR-0004 (utc_apply_bus ICD) before any UTC-scheduled-parameter MS.

## Re-seed block

GOAL: Author the core ICDs (streaming_bus, register_bus, obg_frame) which are referenced by every MS and must be frozen before MS authoring begins.

CONSTRAINTS:
- Target device: AMD XCZU43DR-2FFVE1156E (Zynq UltraScale+ RFSoC, speed grade -2)
- 5-doc framework (ADR-0001) and MS/uArch split (ADR-0005, pending Gastón confirmation)
- FAD §4 v0.5 baseline: 4 clock domains (clk_aurora_user / clk_dsp / clk_rfdc_axis / clk_mgmt); single-clock filter-bank wrappers; clk_dsp ↔ clk_rfdc_axis synchronous-derived 1:3
- Control-path-only reset policy (FAD-DEC-16, hard rule R1–R6) — every MS conforms
- ADR-0006 (FFT4096 pair) and ADR-0007 (single-bitstream sample-rate switching) drive clocking; both proposed, pending OQ-§4-4 and OQ-§4-9 resolution

PRIOR CONCLUSIONS:
- FAD §4 v0.5 ready for FAD.md integration; 13 CDC entries; 4 resets
- ICD authoring order: streaming_bus → register_bus → obg_frame; ADR-0004 (utc_apply_bus) triggered before first UTC-scheduled-parameter MS
- streaming_bus is most referenced (every datapath block) — define AXI4-Stream payload format, TUSER sideband fields, backpressure rules first
- register_bus likely AXI-Lite over clk_mgmt; CDC handled at consumer side per CDC-06/CDC-07
- obg_frame is the OBG Aurora frame protocol carried over clk_aurora_user; informs obg_link MS

CURRENT QUESTION: Decide ICD authoring sequence and start with streaming_bus. Confirm whether to trigger ADR-0004 (utc_apply_bus ICD) in parallel, or sequence it after the three core ICDs.

## Artefacts produced

- `bpms-sfu-fpga-design/arch/fad/FAD.md` §4 (drop-in replacement, FAD v0.5) — content sourced from `fad_section4_v05.md` produced this chat. Replaces lines 337–367 of FAD.md v0.4.
- §4 supplement (`docs/working/fad-section-4-supplement.md`) and ADRs 0006/0007 (`arch/adr/0006-filter-bank-engine-selection.md`, `arch/adr/0007-asic-fpga-sat-sample-rate-switching.md`) consumed as inputs — already filed from prior chats; no new copies emitted.
