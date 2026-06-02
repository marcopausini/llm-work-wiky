# Next chat prompt — FAD §3 Dataflow + §4 Clocking and Reset Architecture

Open a new chat in the **bpms-sfu-architecture** Claude Project and paste the block below.

---

I am continuing the FAD authoring for the BPMS 1.0 Satellite Frontend Unit (SFU). FAD §1, §2, §6 are committed. This chat tackles the next phase-1 sections: §3 Dataflow and §4 Clocking & Reset Architecture, and produces ADR-0002 (clock architecture) as `proposed`.

The deliverable of THIS chat is:
1. FAD §3 Dataflow populated (narrative DL + UL + debug, with sample rates, bus widths, frame boundaries per stage)
2. FAD §4 Clocking and Reset Architecture populated (§4.1 clock inventory, §4.2 clock topology, §4.3 primary vs secondary mode comparison, §4.4 reset topology, §4.5 CDC inventory)
3. ADR-0002 drafted as `proposed` covering: CDC FIFO placement (resolves state.md flag #1), primary/secondary clock mode topology, reset architecture rationale

## Working principles for this chat

1. **Read SFU-001 first.** §6.1 (DL CDC), §7.5 (UL OBG output), §11 (timing & sync), §6.4/§7.4 (filter bank internal CDC at 341.333/320 MHz boundary), §9.1 (RFSoC platform), §12.2 (sample rates). Cite sections inline. Don't guess where SFU-001 leaves a structural choice open — surface options and recommend.

2. **§4.5 CDC inventory is load-bearing.** Every CDC in the design MUST appear in this table, with ID, from-domain, to-domain, type (sync/async/elastic), data width, mechanism (handshake/gray-code/async FIFO/etc.), and MDS pointer. Cross-check against the 31-module inventory in FAD §6 — every module touching multiple clock domains must be represented.

3. **Sample-rate scaling.** SFU-001 §12.2 specifies FPGA-satellite mode uses ×1.024 sampling rate (4096 → 4194.304 Msps). Every sample-rate-derived value in §3 must note both ASIC-sat and FPGA-sat values, or use a unified notation.

4. **Resolve flag #1 in state.md.** The `cdc_fifo_dl` placement (between `obg_rx` and `obg_phase_align` vs other) and `cdc_fifo_ul` placement (between `bins_sel_ul` and `obg_tx`) must be decided here. Justify in §4.5 and capture the decision in ADR-0002.

5. **Two-mode topology.** Primary mode: each SFU's CDR locks to one pre-selected OBG DL Aurora lane; recovered clock drives DSP and RFdc. Secondary mode: external 10 MHz at CI port → SFU PLL → DSP/RFdc; OBG UL driven from SFU clock back upstream. Both paths must appear in §4.2/§4.3 and the CDC inventory must accommodate both modes.

6. **Don't redecompose.** The 31-module inventory and three-view §2 are frozen. Add new modules only if §3/§4 reveals a structural gap; raise as an open question first.

## Source documents

- **Primary:** `BPMS_1.0_SFU_Architecture_v1.6.docx` (BPMS-1.0-SFU-001) — §6.1, §6.4, §7.4, §7.5, §9.1, §11, §12.2
- **Secondary:** `BPMS_1.0_Architecture_Document_v2.4.docx` (BPMS-1.0-ARCH-001) — §11 (system clock distribution), §12 (interfaces), §15.2 (UTC apply mechanism)

Both uploaded to project KB.

## Constraints (unchanged from prior chats)

- Target: BittWare RFX-8440A (AMD XCZU43DR-2FFVE1156E, speed grade -2)
- Framework is 5-doc Markdown (FAD/MDS/ICD/ADR/RTM)
- Repo `bpms-sfu-fpga-design/`, flat layout
- Citation discipline mandatory; placeholder markers `[TBD]` / `[STUB]` / `[ASSUMPTION]` / `[INFERRED]` greppable
- `state.md` is canonical
- Module naming convention frozen (no `sfu_` prefix; whitelist abbreviations; DL/UL suffix policy)
- 31-module inventory frozen — see FAD §6

## Suggested order of work

1. Read SFU-001 §6.1, §6.4, §7.4, §7.5, §9.1, §11, §12.2. List every clock domain mentioned or implied. Cite section for each. Present the list before drafting §4.1.
2. Propose the §4.1 clock inventory table. Get approval.
3. Propose the §4.5 CDC inventory table. For each CDC, identify: source module, sink module, mechanism, data width, sync/async classification. Get approval.
4. Resolve flag #1 — decide CDC FIFO placement on DL and UL. Surface options, recommend, get approval.
5. Draft §4.2 clock topology (Mermaid diagram) showing primary and secondary mode paths.
6. Draft §4.3 primary vs secondary mode comparison table.
7. Draft §4.4 reset topology — per-domain reset trees, async assert / sync deassert convention, reset-during-mode-switch behaviour.
8. Draft §3 dataflow narrative, stage by stage, leveraging §4 clock domains and sample rates.
9. Draft ADR-0002 covering: CDC placement decisions, clock-mode topology, reset architecture.
10. Update state.md: flag #1 resolved → add to "decisions made"; ADR-0002 status `proposed`; current phase update.

Open questions, options, and recommendations should be visible in the chat, not buried in the diagram.

## Prior chat context (load-bearing)

- DL ordering: `obg_rx → cdc_fifo_dl → obg_phase_align → bins_sel_dl → filter_bank_synth → cw_beacon (sum) → band_gain_dl → band_doppler_dl → rfdc_wrap → rf_port_sel → T0/T1/T2`
- UL ordering: `R0/R1/R3 → rf_port_sel → rfdc_wrap → band_doppler_ul → band_rms_det_ul → band_gain_ul → filter_bank_analysis → bins_sel_ul → cdc_fifo_ul → obg_tx`
- Sub-band A/B handled inside each block, not split at top level
- PS-PL split: PS hosts GEM, mgmt protocol stack, SGP4 propagation. PL hosts `mgmt_if` (AXI bridge), `reg_bus`, `tle_compute` PL portion (1PPS-edge NCO load)
- Clock domain shorthand from FAD §6: `aurora_rx`, `aurora_tx`, `dsp`, `dsp_in`, `dsp_out`, `rfdc`, `mgmt`, `ps_axi`
- Filter bank IP internal CDC: 341.333 MHz input → 320 MHz output (DL synth) and reverse for UL analysis (per SFU-001 §6.4 / §7.4)
- ADC/DAC: 4096 Msps (ASIC sat) / 4194.304 Msps (FPGA sat); FPGA-sat = ×1.024 scaling

After this chat closes: next chat = FAD §5 Memory Architecture + Core ICDs (streaming_bus, register_bus, obg_frame).
