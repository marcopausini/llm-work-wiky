---
id: 2026-04-25--bpms-sfu-architecture--fad-section-2-and-6-bootstrap
date: 2026-04-25
project: bpms-sfu-architecture
type: decision
status: resolved
topics: [fad, module-inventory, naming-convention, mermaid-diagrams, sfu-signal-chain, ps-pl-split]
source_chat: claude-opus-4.7
---

# FAD §2 and §6 bootstrap — top-level block diagram and 31-module inventory

## Context
First pass through SFU-001 v1.6 to derive the SFU FPGA top-level block diagram and seed the module inventory. Repo `bpms-sfu-fpga-design/`, framework already settled (5-doc Markdown, ADR-0001 proposed). Previous state: empty FAD skeleton with §1.2 boundary stubs.

## Key findings
- SFU-001 yields ~25 functional blocks across §4–§8, §11, §14–§17. Three additional blocks are inferred and structurally required (`cdc_fifo_ul`, `reg_bus`, `nv_cfg`). Final count after MDS-time decomposition: 31 modules including `sfu_top`.
- The DL signal chain ordering `filter_bank_synth → cw_beacon (sum) → band_gain_dl → band_doppler_dl` is unambiguous from §6.8 ("after F.B. Synthesis and BEFORE Band Doppler and Band Gain"). The `+` summation node is an injection mechanism, not a separate module.
- UL RMS detector (`band_rms_det_ul`) sits **before** Band Gain UL per §14.2. DL RMS detector sits **after** Band Gain DL. Asymmetry reflects loop topology — DL RMS measures actuated output for closed-loop control; UL RMS measures received level for telemetry reference.
- Sub-band A/B parallelism is NOT split at top level. Each affected RTL module handles both sub-bands internally (filter bank, gain, Doppler, RMS, bin selector). May resolve to two parameterized instances at MDS time.
- `rfdc_wrap` encapsulates the AMD RFdc tile (ADC+DAC+DUC+DDC+NCO) as one PL block; `rf_port_sel` is a separate logical port-MUX block with its own UTC-scheduled control. Decoupling justified by independent control semantics.
- `band_rms_det_dl/ul` are first-class RTL blocks (not embedded in `band_gain_dl/ul`) because they serve both telemetry and Band Gain loop feedback — telemetry function decoupled from actuation function.
- `tle_compute` and `mgmt_if` are PS-PL split blocks. PS hosts SGP4 propagation, GEM, mgmt protocol stack; PL hosts AXI bridge, register fabric, 1PPS-edge NCO load. PS software is OUT OF SCOPE of `bpms-sfu-fpga-design`.
- `cdc_fifo_dl` and `cdc_fifo_ul` placement is INFERRED from §6.1 / §7.5 — neither section states the crossing point explicitly. Both placed at the OBG-Aurora boundary on the assumption that bin-domain logic runs on DSP clock.

## Decisions
- **Module naming convention** — no `sfu_` prefix (repo is SFU-scoped); `sfu_top` sole exception. DL/UL suffix only when same function exists in both directions as separate RTL modules. Whitelist abbreviations: `obg`, `dl`, `ul`, `rx`, `tx`, `sel`, `synth`, `mgmt`, `if`, `rfdc`, `cdc`, `pps`, `rms`, `tle`, `cw`. Spell out everything else (`band_gain` not `bg`, `filter_bank` not `fb`).
- **Three-view organization for FAD §2** — DL datapath, UL datapath, infrastructure (mgmt + timing + shared). Clocks deferred to FAD §4 with their own diagram.
- **Authoritative diagrams live in `arch/fad/diagrams/`**, condensed in-FAD copies in §2.1/§2.2/§2.3 for self-contained readability. Risk of drift acknowledged; mitigated by cross-references.
- **PS scope boundary captured** in FAD §1.2.2 and state.md — PS software (TLE/SGP4, mgmt protocol, GEM driver) is out of scope. Triggers ADR-0003.
- **Spur detection vs spectrum monitor** drawn as separate annotated taps at the same point. Will merge if OI-S10 resolves "shared HW resource."
- **Static configuration mode** is a register bit in the affected blocks (`band_doppler_dl/ul`, `band_gain_dl/ul`), not a separate block.

## Open questions
- **Flag #1** — `cdc_fifo_dl` placement (before vs after `obg_phase_align`); `cdc_fifo_ul` placement (before vs after `bins_sel_ul`). Resolves at FAD §4 / ADR-0002.
- **Flag #5** — `utc_apply_engine` centralized vs distributed shadow registers. Deferred to first scheduled-apply MDS / ADR-0004.
- **Flag #6** — NV-restore-on-POR FSM placement (`mgmt_if` vs dedicated `por_fsm` vs inside `reg_bus`). MDS-time decision.
- **Flag #10** — `cw_beacon` injection sum node — inside `cw_beacon` module or as combinational sum at `band_gain_dl` input. MDS-time decision.
- **Flags #11, #12** — DL capture tap B (post `band_doppler_dl`) and UL capture tap D (post `bins_sel_ul`) are `[ASSUMPTION]` pending OI-S08 resolution.

## Action items
- [ ] User reviews FAD draft and commits to `bpms-sfu-fpga-design/arch/fad/FAD.md` and `arch/fad/diagrams/{dl-datapath,ul-datapath,infrastructure}.md`
- [ ] Open new chat for FAD §3 + §4 + ADR-0002 (per re-seed block below)
- [ ] After Chat A closes: open new chat for FAD §5 + Core ICDs
- [ ] After phase 1 closure: schedule chat for FAD §7–§11
- [ ] After phase 2 closure: ADR-0003 (PS-PL partitioning) then first MDS (`rf_port_sel`)

## Artefacts produced

- `projects/bpms-sfu-architecture/state.md` — updated with 31-module inventory, three-view organization, PS scope boundary, 12 active flags, ADR pipeline, next-phase plan
- `bpms-sfu-fpga-design/arch/fad/FAD.md` v0.1 — §1, §2, §6 populated; §3–§5, §7–§14 remain skeleton or seeded
- `bpms-sfu-fpga-design/arch/fad/diagrams/dl-datapath.md` — authoritative DL datapath diagram with full annotations
- `bpms-sfu-fpga-design/arch/fad/diagrams/ul-datapath.md` — authoritative UL datapath diagram with full annotations
- `bpms-sfu-fpga-design/arch/fad/diagrams/infrastructure.md` — authoritative infrastructure / shared view diagram

## Re-seed block

GOAL: populate FAD §3 Dataflow and §4 Clocking & Reset Architecture; draft ADR-0002 (clock architecture) as `proposed`
CONSTRAINTS:
- Target: BittWare RFX-8440A, AMD XCZU43DR-2FFVE1156E, speed grade -2
- 5-doc Markdown framework (FAD/MDS/ICD/ADR/RTM); flat repo `bpms-sfu-fpga-design/`
- `state.md` is canonical; SFU-001 v1.6 primary source, ARCH-001 v2.4 secondary
- Citation discipline mandatory; placeholder markers `[TBD]`/`[STUB]`/`[ASSUMPTION]`/`[INFERRED]` greppable
- §4.5 CDC inventory must list every clock-domain crossing — no CDC may exist outside this table
- DL/UL ordering and 31-module inventory from FAD §6 are LOAD-BEARING — do not redecompose
PRIOR CONCLUSIONS:
- Module naming convention frozen (no `sfu_` prefix; whitelist abbreviations)
- Three-view §2 organization frozen (DL / UL / infrastructure)
- 31 modules in §6 inventory; clock domain shorthand defined: `aurora_rx`, `aurora_tx`, `dsp`, `dsp_in`, `dsp_out`, `rfdc`, `mgmt`, `ps_axi`
- PS-PL split confirmed: PS does SGP4/GEM/mgmt protocol; PL hosts `mgmt_if`, `reg_bus`, `tle_compute` PL portion
- `cdc_fifo_dl` and `cdc_fifo_ul` placement is INFERRED, requires resolution in §4 and ADR-0002
- Sample rates per stage: ADC/DAC 4096 Msps (ASIC sat); filter bank input 341.333 MHz, output 320 MHz (per SFU-001 §6.4); FPGA-sat ×1.024 scaling factor on all sample-rate-derived params
- Primary clock: CDR from pre-selected OBG DL Aurora lane; Secondary clock: external 10 MHz at CI port; switchover policy in §11.4
- Clock mode is per-SFU UTC-scheduled at commissioning, stored in `nv_cfg`
CURRENT QUESTION: how should the SFU clock domains be inventoried, where do the CDC FIFOs sit, and what reset topology supports primary/secondary mode switching without disturbing in-flight samples
