---
id: 2026-05-07--bpms-sfu-architecture--filter-bank-engine-and-sample-rate-switching
date: 2026-05-07
project: bpms-sfu-architecture
type: decision
status: resolved
topics: [filter-bank, mdft, fft4096, flexfft240, rfdc, sample-rate, asic-sat, fpga-sat, clocking, cdc, adr]
source_chat: claude-opus-4.7
---

# SFU filter bank engine selection and ASIC-sat / FPGA-sat sample-rate switching

## Context

Investigation of the CPBF B2D filter-bank IP (Filter Bank Design Specification v0.1, Jul 2025) for reuse in the SFU FPGA design. CPBF B2D System Architecture Document v0.8 reviewed for context. SFU-001 v1.6 §6.4/§7.4 currently describes FLEXFFT240 — flagged for reconciliation. The investigation produced two coupled architectural decisions: which CPBF filter-bank pair the SFU should reuse, and how to support both satellite types (ASIC-sat 4,096 MSPS / FPGA-sat 4,194.304 MSPS) on the same hardware.

## Key findings

- The CPBF filter-bank IP exposes 4 blocks organised as 2 analysis/synthesis pairs distinguished by their FFT engine — not by direction. CPBF "DL/UL" naming is satellite-payload convention (DL = sat-to-ground); SFU role mapping flips.
- **FFT4096 pair**: `analysis_dnlink_filterbank` (wideband → 12,288 bins, polyphase pre-filter + FFT4096) and `synthesis_uplink_filterbank` (12,288 bins → wideband, IFFT4096 + polyphase post-filter). Single 341.333 MHz clock. No per-beam-slot structure.
- **FLEXFFT240 pair**: `synthesis_dnlink_filterbank` and `analysis_uplink_filterbank`. Use FLEXFFT240 (5 modes: 240/120/60/40/20-pt) per beam slot. 48 beam slots × per-slot FFT-size config. Internal 320↔341.333 MHz CDC.
- Numerical parameters of the IP match the SFU ASIC-sat case exactly (1,024 MHz native sub-band BW, 12,288 bins at 83.333 kHz, 3 lanes, 18-bit IQ, 341.333 MHz DSP).
- FPGA-sat scales the entire sample-rate tree by ×1.024 (4,194.304 MSPS, 1,048.576 MHz BW, 85.332992 kHz bin width, 349.525 MHz DSP). FFT structure, lane count, bin count, and polyphase prototype taps (normalised-frequency design) are unchanged.
- **FPGA-sat 2.56 MHz beam mode (~30 bins) is not in FLEXFFT240's mode table.** FLEXFFT240's 5-mode bypass schedule (radix `5×4×3×2×2`) supports only 240/120/60/40/20-pt FFTs. 30-pt would need a different decomposition (e.g. `5×3×2`) — not a small modification.
- FFT4096 pair has no per-beam structure → unaffected by FPGA-sat beam BW.
- XCZU43DR Gen 3 RFdc supports both 4,096 and 4,194.304 MSPS (well within Gen 3 sample-rate envelope).
- RFdc tile PLL is runtime-reconfigurable via `XRFdc_DynamicPLLConfig`. Switch sequence is not hitless: stop tile, reprogram PLL, recalibrate, MTS re-sync (if used), restart. Total RF-disabled window ~tens of ms to ~1 s.
- ARCH-001 §7.4 / SFU-001 §4.2 explicitly state satellite-type changes require planned re-commissioning — live switching not supported. Confirmed compatibility with the runtime cost of the switch.
- Aurora SERDES line rate (19.2 Gbps) is fixed by OBG ICD — independent of satellite type. Aurora link stays up across the switch.
- Cleanest DSP clock topology: tie PL DSP MMCM input to RFdc tile streaming-AXIS reference clock (= sample_rate / DDC = 1,024 / 1,048.576 MHz). Fixed M/D ratio from MMCM produces 341.333 / 349.525 MHz automatically. No DRP, no second MMCM, no BUFGMUX. ×1.024 scaling propagates structurally.
- DRP MMCM reconfig held as fallback if Option D-1 fails timing or VCO-range check.
- Items that scale with sample rate: ADC/DAC rate, sub-band BW, bin width, DSP clock, Doppler NCO frequency words, FFT frame timing.
- Items that do NOT scale: filter-bank prototype taps, FFT structure, bin count (12,288), Aurora rate, all CSR layouts, resource utilisation.
- Eliminating the FLEXFFT240 internal 320↔341.333 MHz CDC by selecting the FFT4096 pair simplifies the SFU CDC inventory by 2 entries (DL + UL).

## Decisions

- **ADR-0006 (proposed):** SFU `filter_bank_synth` wraps CPBF `synthesis_uplink_filterbank` (IFFT4096 + polyphase post-filter); SFU `filter_bank_analysis` wraps CPBF `analysis_dnlink_filterbank` (polyphase pre-filter + FFT4096). FLEXFFT240-based pair rejected on architectural cleanness + FPGA-sat 30-bin gap.
- **ADR-0007 (proposed):** Single FPGA bitstream supports both satellite types. RFdc tile PLL runtime-reconfigurable (DynamicPLLConfig). DSP MMCM derives from RFdc streaming-AXIS clock; ratio fixed; output scales by ×1.024 automatically. Aurora link preserved across switch. DSP domain reset across the reconfiguration window. Re-commissioning state machine in mgmt plane drives the sequence.
- Both ADRs are linked: ADR-0006's FFT4096 selection is rate-agnostic by design and enables ADR-0007's single-bitstream model.

## Open questions

- SFU-001 v1.6 §6.4/§7.4 describes FLEXFFT240 — confirm with system architect whether deliberate or copy-paste from CPBF spec; ADR-0006 acceptance gated on resolution.
- BittWare RFX-8440A LMK/LMX clocking subsystem capability for 1.024-scalable reference output — confirm with HW lead and BittWare datasheet.
- MMCM VCO range validity at both `clk_rfdc_axis` values × chosen M — IP customisation check.
- Multi-Tile Sync usage decision — affects switch sequence step S5.
- CPBF reference-model availability and reuse scope (parameterised on sample rate?) — confirm with CPBF DSP team.
- IP version pinning + change-control mechanism if CPBF revises the IP.
- OBG bandwidth budget at FPGA-sat full bin density — system-level concern flagged for system architect.

## Action items

- [ ] Raise SFU-001 §6.4/§7.4 wording inconsistency with system architect (Shlomi); resolve before ADR-0006 acceptance
- [ ] Confirm RFX-8440A LMK/LMX 1.024-scalable reference capability with HW lead (OI-RFX-001)
- [ ] Confirm MTS usage decision with system architect (OI-MTS-001)
- [ ] Confirm CPBF reference-model availability and parameterisation with CPBF DSP team (FF)
- [ ] Update FAD §4 using `fad-section-4-supplement.md` content
- [ ] Update FAD §6 inventory: classify `filter_bank_synth` and `filter_bank_analysis` as `wrap` of CPBF blocks; document CPBF block names in MS body
- [ ] Update FAD §1.4 FAD-DEC list: add filter-bank engine selection (ADR-0006) and sample-rate switching (ADR-0007) entries
- [ ] Update §13.1/13.2 open issues list with OQs from both ADRs
- [ ] Schedule OOC synthesis run for filter-bank IP at 349.525 MHz on XCZU43DR-2 to validate FPGA-sat timing closure (RTL designer, post Task 8)

## Artefacts produced

- `bpms-sfu-fpga-design/arch/adr/0006-filter-bank-engine-selection.md` — ADR-0006 (proposed) selecting FFT4096 pair from CPBF for SFU filter-bank wrappers.
- `bpms-sfu-fpga-design/arch/adr/0007-asic-fpga-sat-sample-rate-switching.md` — ADR-0007 (proposed) defining single-bitstream sample-rate switching topology and sequencing.
- `bpms-sfu-fpga-design/docs/working/fad-section-4-supplement.md` (or merge directly into FAD §4) — supplemental input for FAD §4 update with sample-rate scaling rule, clock inventory entries, derivation chain, reset topology, CDC inventory, and switch sequencing pointer.

## Re-seed block

GOAL: Author FAD §4 (Clocking and Reset Architecture) and the two filter-bank module specs (`filter_bank_synth`, `filter_bank_analysis`) for the SFU FPGA on BittWare RFX-8440A.

CONSTRAINTS:
- Target device: AMD XCZU43DR-2FFVE1156E (Zynq UltraScale+ RFSoC, speed grade -2)
- Two satellite-type sample rates: 4,096 / 4,194.304 MSPS (×1.024 scaling)
- Single FPGA bitstream serves both types
- Aurora SERDES rate fixed by OBG ICD (19.2 Gbps × 4 lanes), independent of sample rate
- Re-commissioning event allowed for satellite-type switch (~tens of ms to ~1 s RF-disabled OK)
- CPBF filter-bank IP reuse mandated by program (Filter Bank Design Specification v0.1)
- ADR-0001 5-doc framework, ADR-0005 MS/uArch split

PRIOR CONCLUSIONS:
- ADR-0006 (proposed): Use CPBF FFT4096 pair (`analysis_dnlink_filterbank`, `synthesis_uplink_filterbank`) for SFU filter banks. Reject FLEXFFT240 pair (per-beam-slot structure not needed at SFU; FPGA-sat 2.56 MHz / 30-bin not in FLEXFFT240 mode table).
- ADR-0007 (proposed): Single bitstream + RFdc DynamicPLLConfig + DSP MMCM ratio-tied to RFdc tile clock (×1.024 propagates structurally). Aurora preserved across switch; DSP domain reset across reconfiguration window. State machine in mgmt plane.
- Filter-bank IP is rate-agnostic (normalised-frequency polyphase taps; fixed FFT structure). Only DSP clock changes.
- SFU-001 v1.6 §6.4/§7.4 describes FLEXFFT240 — must be reconciled with system architect; treated as inconsistency for now.

CURRENT QUESTION: Author FAD §4 in full using the supplement file as input. Then start `filter_bank_synth` and `filter_bank_analysis` MSs by cloning `arch/modules/_template.md` (subject to Gastón's confirmation per ADR-0005). Confirm RFX-8440A LMK/LMX capability with HW lead before fixing the clock topology.
