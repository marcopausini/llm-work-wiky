# Re-seed prompt — FAD §4 Clocking and Reset Architecture

## Where we left off

FAD v0.4 is complete through §1 (scope/boundary), §2 (three `.drawio` diagram views with inline dataflow narrative), and §6 (23 MS files, ~28 RTL instances). §3 is collapsed (content absorbed into §2). §4–§12 remain skeleton templates.

All three diagrams (top datapath, sub-band processing, infrastructure) verified correct against ARCH-001 v2.4 and SFU-001 v1.6. Infrastructure `.drawio` had 7 layout issues — all fixed. Mermaid retired from §2.1/§2.2.

## Task for this chat

**Author FAD §4 — Clocking and Reset Architecture.** This is the next section in dependency order and a prerequisite for G1 (Orient) gate closure. §4 feeds directly into every MS's "Clocks and resets" table and the CDC inventory referenced by the execution-flow CDC gate.

### §4 sub-sections to populate

1. **§4.1 Clock inventory** — table of every clock domain in the design with name, frequency, source, consumers, and notes. Clock domain shorthand from §6 (`aurora`, `dsp`, `dsp_in`/`dsp_out`, `rfdc`, `mgmt`, `ps_axi`) is the starting point but needs full population. The values `dsp_in`/`dsp_out` are taken from the current implementation of the filter banks modules, but it should be verified if the same frequencies are needed in this project

2. **§4.2 Clock topology** — PLL/MMCM sources, buffers, gated vs free-running. The RFSoC XCZU43DR has specific clocking resources; the RFX-8440A board constrains which reference clocks are available. Primary clock mode recovers from OBG Aurora CDR; secondary from external 10 MHz at CI port.

3. **§4.3 Primary vs Secondary clock mode** — per SFU-001 §11.1–§11.4: primary (OBG CDR → drives everything), secondary (10 MHz CI → PLL → drives everything, propagates upstream to DCU via OBG UL). Switchover policy: auto-switch to secondary on CDR unlock, manual recovery.

4. **§4.4 Reset topology** — reset names, domains, assertion/release rules, scope. The RFSoC has PS-driven resets, PL fabric resets, and GT/RFDC-specific resets.

5. **§4.5 CDC inventory** — every clock-domain crossing in the design. No CDC may exist that is not in this table. Must be bidirectionally consistent with `report_cdc` output at sign-off (per 04-execution-flow.md §11). Approved mechanisms: two-flop synchroniser, async FIFO with Gray-code pointers, handshake, vendor IP bus-synchroniser.

### Source material to extract from

- **SFU-001 §11.1–§11.5**: Primary/secondary clock modes, 1PPS, inter-SFU phase alignment
- **SFU-001 §6.1**: CDR clock recovery on OBG DL lanes
- **SFU-001 §6.4**: Filter bank processing clocks (341.333 / 320 MHz, CDC boundaries inside IP)
- **SFU-001 §7.5**: OBG UL clock propagation in secondary mode
- **SFU-001 §9.1**: RFSoC ADC/DAC sampling rate (4,096 Msps ASIC / 4,194.304 Msps FPGA)
- **SFU-001 §12.2**: FPGA satellite ×1.024 scaling
- **ARCH-001 §11.1–§11.5**: System-level clock distribution
- **ARCH-001 §12.1**: SFU interface table (ET, CI, CO, Q0 ports)
- **FAD §6**: Module inventory with clock domain assignments per block
- **FAD §2.3 infrastructure diagram**: `clock_ctrl` and `pps_capture` blocks

### Key constraints and known issues

- CDC methodology finalised: FAD §4.5 owns project-wide policy (clock relationships, approved mechanisms, intentional boundaries); Vivado `report_cdc` handles per-flop enumeration at CI level.
- Clock-domain relationship classification (synchronous-derived integer-ratio vs. asynchronous) is architecture-layer because it constrains synchroniser choice and resource budgets.
- ADR-0002 (clock architecture) is TBD — this §4 authoring session may produce the content that becomes that ADR, or it may identify the specific decisions that need ADR treatment.
- `dsp_in`/`dsp_out` (341.333 / 320 MHz) are filter-bank IP internal CDC boundaries — need to determine if these are exposed at the module boundary or encapsulated inside the IP wrapper.
- `rfdc` domain (4,096 Msps) relationship to `dsp` domain needs explicit classification.
- `mgmt` clock frequency is TBD — needs a decision or at least a constrained range.
- `ps_axi` clock relationship to `mgmt` clock needs classification.

### Deliverable

Updated FAD §4 (all five sub-sections populated or explicitly TBD with owner) ready for integration into FAD v0.4. Identify any ADR triggers. Update the §6 clock domain shorthand table if §4 authoring reveals additional domains or renames.

## Files to have in context

- `FAD.md` (v0.3 — the version produced last session)
- `BPMS_1.0_SFU_Architecture_v1.6.docx` (SFU-001)
- `BPMS_1.0_Architecture_Document_v2.4.docx` (ARCH-001)
- `04-execution-flow.md` (for CDC gate requirements in §11)
- `05-signoff-criteria.md` (for CDC sign-off criteria in §7)
