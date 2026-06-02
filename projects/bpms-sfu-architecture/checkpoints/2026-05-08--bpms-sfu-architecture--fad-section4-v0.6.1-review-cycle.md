---
id: 2026-05-08--bpms-sfu-architecture--fad-section4-v0.6.1-review-cycle
date: 2026-05-08
project: bpms-sfu-architecture
type: decision
status: ongoing
topics: [clocking, reset, cdc, fad-section-4, reviewer-round, solution-b, si5394, lmk04821, lmx2594, nco-apply-semantics, fifo-empty-state, shadow-active-atomic]
source_chat: claude-opus-4.7
---

# FAD §4 v0.6 rewrite + reviewer round 1 + v0.6.1 patch

## Context

Triggered by FAD §4 review blocker: prior v0.5 had inverted clock chain (LMK as master) contrary to SFU-001 v1.6 §11.1 (recovered clock drives all SFU FPGA logic and ADC/DAC converters). Carry-over from prior session (`2026-05-08--bpms-sfu-architecture--sfu-clock-topology-review`) established the three-stage Si5394 → LMK04821 → LMX2594 ×2 board chain from RFX-8440A v2.2 datasheet + Hardware Reference Guide. This session rewrote the clocking half of §4, applied consistency edits across the remainder, and closed reviewer round 1 with a v0.6.1 delta patch.

## Key findings

- §4.2 framing decision settled: **Solution B canonical** (OBG-recovered as master timing source; Si5394 jitter cleaner is downstream slave), `[ASSUMPTION]` tied to OQ-§4-10 (Shlomi to confirm literal vs loose reading of SFU-001 §11.1). Solution A (board OCXO master with OBG slow discipline) noted as alternative tuning point on the same hardware; final A-vs-B choice captured in ADR-0002 when authored.
- ASCII clock-tree drawings (former §4.2.1 / §4.2.2 / §4.2.3 in v0.5) retired in favour of two `.drawio` figure references (Figure 4.1 — board clock tree; Figure 4.2 — FPGA domains + CDC), with prose narrative replacing the drawings. Consistent with FAD-DEC-02 and aligned with existing diagram convention (`top_datapath`, `subband_processing`, `infrastructure`).
- Sample-rate switching scope: only LMK04821 + LMX2594 + RFdc-tile-PLL chain is reprogrammed; Si5394 outputs to MGT TX refclks are sample-rate-invariant by board topology. This is the structural reason the Aurora link is preserved across the switch — not an ADR-0007 stipulation. Promoted to §4.0 sample-rate scaling table as an invariant row.
- Frequency coherence vs phase coherence split (§4.3.5): frequency coherence between G-SFU SFUs is inherent via common DCU-sourced OBG clock; phase coherence is NOT inherent — deterministic-but-nonzero residual skew from differential Polatis fibre length, GT CDR static offset, on-chip clock-tree delay. Per-SFU phase-trim register architecturally required (OI-S02 / SYS-CLK-09). SFU-001 §11.5 "phase coherence is inherent" wording flagged to Shlomi as misleading (frequency-only).
- Two-stage GPS holdover documented: Si5394 48 MHz XAB crystal (digital memory of last frequency) + LMK04821 VCXO (analogue PLL holdover). Holdover budget at SFU level is TBD pending OQ-§4-11 (inherits ARCH-001 v2.4 OI-013).
- FAD-DEC-17 (`clk_dsp` ↔ `clk_rfdc_axis` synchronous-derived 1:3 via MMCM) downgraded from confirmed to **proposed**. `[ASSUMPTION]` marker carried in §4.0, §4.1, §4.2.2, §4.2.5, §4.2.7, §4.5.2, §4.5.3 footnote, and §4.9 row. Expiry: ADR-0002 + OQ-§4-3 (MMCM VCO range valid at both rates × chosen M).
- Reviewer round 1 outcome: 10 accepted / 3 rejected (1, 8, 13). All rejected items already addressed in v0.6 prior to review — reviewer was reading v0.5 wording or wanted stronger marker discipline than the project convention (one anchor + cross-refs).
- Major architectural refinements from reviewer adoption: (a) NCO no-reset rule scoped to fabric reset only — operational apply semantics at 1PPS / sample-rate switch / CSR write deferred to `band_doppler` and `cw_beacon` MS contracts; (b) FIFO state at sample-rate switch expressed as architectural requirement (empty-state at end of switch) rather than pointer-mechanics prescription (uArch scope); (c) CDC-08 elevated to shadow-active atomic update across all 4 Band Doppler NCO instances at 1PPS edge — same pattern as `utc_apply_engine` for UTC-scheduled.

## Decisions

- Solution B canonical in §4.2; ADR-0002 to finalise after OQ-§4-10 resolution.
- FAD-DEC-17 status: proposed (was: confirmed in v0.5 draft).
- FAD-DEC-15 row credits Si5394 → MGT TX refclks structural invariance for Aurora link preservation across sample-rate switch (not an ADR-0007 stipulation).
- §6 clock-domain shorthand updated: retire `dsp_in` / `dsp_out` (ADR-0006 single-clock filter-bank IP) and `ps_axi` (FAD-DEC-11 collapsed into `clk_mgmt`). Canonical: `aurora` / `dsp` / `rfdc` / `mgmt`.
- 1PPS is an asynchronous edge event, NOT a timing clock. Illustrative XDC in §4.2.7 drops `create_clock pps_in`; only `set_false_path -from [get_ports pps_in]` retained. `pps_capture` owns synchronisation.
- CDC-03 / CDC-04 (Aurora status/control multi-bit) carry an explicit per-bit quasi-static contract annotation, not implicit.
- §4 v0.6.1 frontmatter bump applies the 10 reviewer-adoption edits on top of v0.6.

## Open questions

- OQ-§4-10 (Shlomi): SFU-001 §11.1 "recovered clock drives all SFU FPGA logic and ADC/DAC converters" — literal (bit-derived) or loose (GPS-traceable)? Gates Solution A vs B canonical choice in ADR-0002.
- OQ-§4-4 (HW Lead, 4 sub-questions): (1) MGT128 RX recovered clock routed to Si5394 input on standard RFX-8440A build? (2) Build-time mux at LMK04821 input selects Si5394 O3? (3) VCXO is the programmable variant (OscFrqCfg = C)? (4) Secondary mode — what drives Si5394 when CI port bypasses it; do MGT TX refclks stay valid via holdover?
- OQ-§4-3 (RTL Designer): MMCM VCO range valid at both `clk_rfdc_axis` × chosen M? Gates FAD-DEC-17 confirmation + ADR-0002.
- OQ-§4-9 (Shlomi): SFU-001 §6.4 / §7.4 FLEXFFT240 vs FFT4096 wording reconciliation. Gates ADR-0006 acceptance.
- OQ-§4-11 (Sys Arch): GPS holdover duration and accumulated phase-error budget at SFU level (two-stage: Si5394 XAB crystal + LMK04821 VCXO). Inherits ARCH-001 v2.4 OI-013.
- OI-S02 / SYS-CLK-09 (Sys Arch): Per-SFU phase-trim register specification. Expiry: `obg_link` MS authoring.

## Action items

- [ ] Apply `fad_section4_v0_6_1_patch.md` to the two v0.6 working files (mechanical, 10 edits)
- [ ] Integrate v0.6 + v0.6.1 into `arch/fad/FAD.md` §4 (Task 22); update §1.4 FAD-DEC list (add 11, 14, 15, 16; 17 as proposed); bump FAD v0.3 → v0.6.1
- [ ] Rename `rfx8440a_sfu_clock_tree_IMPROVED.drawio` → `arch/fad/diagrams/clock_tree.drawio` and commit (matches `top_datapath` / `subband_processing` / `infrastructure` naming)
- [ ] Author formal reviewer-rejection response for items 1, 8, 13 per architect-workflow §6.6 protocol; close review round 1 of FAD §4 (Task 25)
- [ ] Trigger ADR-0002 (clock architecture) — gated on OQ-§4-10 + OQ-§4-4 + OQ-§4-3
- [ ] Trigger ADR-0004 (utc_apply_bus ICD) before first UTC-scheduled-parameter MS
- [ ] Flag SFU-001 §11.5 "phase coherence is inherent" wording to Shlomi (frequency-only; phase requires per-SFU trim per OI-S02)
- [ ] Confirm Si5394 three-stage chain naming throughout §4 at FAD.md merge (no "LMK/LMX" left)

## Artefacts produced

Three working drafts produced this session; all currently outside the repo. Eventual integration target: `bpms-sfu-fpga-design/arch/fad/FAD.md` §4.

- `fad_section4_v06.md` — drop-in replacement for §4.0–§4.3.5 (clocking only). OBG-recovered → Si5394 → LMK04821 → LMX2594 three-stage chain; Solution B canonical with `[ASSUMPTION]`; §6 clock-domain shorthand updated; ASCII drawings retired in favour of Figure 4.1 / 4.2 (`.drawio`) references. Working location: outside repo.
- `fad_section4_remainder_v06.md` — drop-in replacement for §4.4–§4.10 (reset + CDC + sample-rate switch + open items + decisions + cross-refs). 10 consistency edits applied to align with §4.0–§4.3.5 v0.6 (Si5394-chain naming, FAD-DEC-17 proposed status, §4.8 open items expanded with OQ-§4-10 / OQ-§4-11 / OI-S02, §4.10 ADR-0006 cross-ref reworded). Working location: outside repo.
- `fad_section4_v0_6_1_patch.md` — 10-edit delta patch on top of v0.6 (original→replacement blocks). Applies reviewer round 1 accepted edits: `clk_dsp` exact values; `clk_aurora_user` heuristic removed; `clk_mgmt` source `[ASSUMPTION]` tied to ADR-0003; XDC drops `create_clock pps_in`; Aurora link drop replaced with TBD per OQ-§4-4 sub-q 4; NCO no-reset clarified (fabric-reset only, apply semantics → MS); power-on time softened to TBD; FIFO empty-state requirement only (pointer mechanics → uArch); CDC-03/04 quasi-static contract annotated; CDC-08 atomic shadow-active update at 1PPS. Working location: outside repo.

Also referenced from prior session, not produced here: `rfx8440a_sfu_clock_tree_IMPROVED.drawio` (uploaded as input this session; awaiting commit as `arch/fad/diagrams/clock_tree.drawio`).

## Re-seed block

GOAL: integrate FAD §4 v0.6.1 into `arch/fad/FAD.md`; bump FAD v0.3 → v0.6.1; commit `clock_tree.drawio` and §1.4 FAD-DEC list updates.

CONSTRAINTS:
- Target device XCZU43DR-2FFVE1156E on BittWare RFX-8440A
- Source-of-truth documents: BPMS-1.0-ARCH-001 v2.4, BPMS-1.0-SFU-001 v1.6
- Solution B canonical, [ASSUMPTION] tied to OQ-§4-10 (Shlomi); Solution A noted as alternative pending ADR-0002
- Three-stage off-FPGA chain: Si5394 (jitter cleaner, MGT TX refclks + LMK04821 ref) → LMK04821 (PLL + clock distribution, VCXO holdover) → LMX2594 ×2 (RF PLLs to RFdc tile sample-clock references)
- Four FPGA clock domains: clk_aurora_user / clk_dsp (341.333333 / 349.525333 MHz) / clk_rfdc_axis (1024 / 1048.576 MHz) / clk_mgmt
- clk_dsp ↔ clk_rfdc_axis synchronous-derived 1:3 via MMCM is `[ASSUMPTION]` pending ADR-0002 + OQ-§4-3
- Sample-rate switching reprograms LMK04821 + LMX2594 + RFdc tile PLL only; Si5394 untouched (preserves Aurora MGT TX refclks)
- Control-path-only reset policy (FAD-DEC-16, hard rules R1–R6); DSP datapath pipeline NOT reset
- 4 resets: rst_mgmt_n, rst_dsp_ctrl_n, rst_aurora_n, rst_rfdc_n
- Reviewer round 1 closed: 10/13 accepted; 3 rejected (1, 8, 13) with rationale in-chat — formal §6.6 response still pending (Task 25)

PRIOR CONCLUSIONS:
- ASCII clock-tree drawings retired; Figure 4.1 (board tree) + Figure 4.2 (domains + CDC) replace them, both in `clock_tree.drawio`
- Frequency coherence between G-SFU SFUs is inherent (common OBG clock); phase coherence is NOT inherent — per-SFU phase-trim required (OI-S02 / SYS-CLK-09)
- Two-stage holdover: Si5394 48 MHz XAB crystal + LMK04821 VCXO; budget TBD per OQ-§4-11
- NCO phase apply semantics belong in `band_doppler` and `cw_beacon` MS contracts — not §4 (§4 owns fabric-reset policy only)
- `obg_link` FIFO must be empty-state at end of sample-rate switch; pointer mechanics are uArch scope
- CDC-08 Band Doppler NCO update is shadow-active atomic at 1PPS edge across all 4 instances (DL/UL × A/B) — same pattern as utc_apply_engine
- CDC-03 / CDC-04 Aurora status/control use per-bit two-flop with explicit quasi-static contract
- 1PPS is asynchronous edge event, not a timing clock — no `create_clock` in XDC
- 1 GbE management interface assumed sourced from PS `pl_clk0` — `[ASSUMPTION]` tied to ADR-0003

CURRENT QUESTION: Mechanical integration of three patch files into `arch/fad/FAD.md`, version bump, drawio commit, and dispatch of OQ-§4-10 / OQ-§4-4 / OQ-§4-9 to their owners (Shlomi / HW Lead / Shlomi). Open: should the formal §6.6 reviewer-rejection response (Task 25) be authored before or after FAD.md merge?
