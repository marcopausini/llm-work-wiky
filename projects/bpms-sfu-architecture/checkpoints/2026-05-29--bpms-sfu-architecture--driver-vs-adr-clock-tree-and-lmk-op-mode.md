---
id: 2026-05-29--bpms-sfu-architecture--driver-vs-adr-clock-tree-and-lmk-op-mode
date: 2026-05-29
project: bpms-sfu-architecture
type: insight
status: open
topics: [adr-0010, adr-0011, lmk04821, lmx2594, clock-tree, jesd204b, sysref, phase-noise, sat-switch]
source_chat: claude-opus-4.7
---

# Driver-vs-ADR clock-tree deltas (BPMS 1.0 + BPMS 0.5) and the NestedZeroDelay question

## Context

BittWare returned driver logs from `rfx_clock_setup.elf` for BPMS 1.0 (4096 and 4194.304 MS/s) and BPMS 0.5 (3932.16 MS/s in both `lmk_op_mode=0` DualLoopMode and `lmk_op_mode=2` NestedZeroDelay). BittWare also flagged in their email that the LMK04821 PLL1 feedback path differs between the two op modes — fPD1 = 30.72 MHz in DualLoop, 10.24 MHz in NestedZeroDelay (PLL1 N divider fed by the 10.24 MHz SYSREF SDCLKout). Goal of the session: compare driver-programmed settings against ADR-0010 (BPMS 1.0 H.1 plan) and ADR-0011 (BPMS 0.5 baseline), recompute the flat-floor phase noise, and assess what this implies for the design.

## Key findings

### Driver-vs-ADR pattern (consistent across BPMS 1.0 and BPMS 0.5)

- **LMK04821 PLL2 — driver halves fPD2 and doubles N_total vs the ADR plan**, same fVCO and same LMK output. Costs +3 dB on the LMK PLL2 floor at the LMK VCO in every configuration examined. ADR-0010 ASIC: ADR fPD2 = 81.92 MHz / N_total = 25 with OSC_2X ON; driver fPD2 = 40.96 MHz / N_total = 50 with OSC_2X OFF. ADR-0010 FPGA: ADR fPD2 = 122.88 MHz / N = 16 / R = 1; driver fPD2 = 61.44 MHz / N = 32 / R = 2. ADR-0011 FPGA: same pattern (ADR R=1, driver R=2).
- **LMX2594 — driver enables `VCO_PHASE_SYNC = 1`** with `IncludedDivide = 4`, in every log. ADRs implicitly assumed `VCO_PHASE_SYNC = 0`. The IncludedDivide forces N_eff ≥ 112 (because PLL_N_programmed ≥ 28) at fVCO ≤ 12.5 GHz, capping fPFD_LMX at ≈ 65 MHz at the BPMS sample rates. ADRs' assumed fPFD values (204.8 / 131.072 / 245.76 MHz) are not realisable with `VCO_PHASE_SYNC = 1` / CHDIV = 2.
- **Net flat-floor at ADC sample clock** (driver vs ADR): BPMS 1.0 ASIC −110.6 vs ADR −113.7 (Δ +3.1 dB); BPMS 1.0 FPGA −112.0 vs ADR −115.0 (Δ +3.0 dB); BPMS 0.5 FPGA (mode 2) −112.5 vs ADR −115.7 (Δ +3.2 dB). LMK floor dominates; the LMX +6 dB self-noise penalty from VCO_PHASE_SYNC is buried by the LMK floor.

### LMK04821 PLL1 in NestedZeroDelay — BittWare's correction

- BittWare's email is correct and confirmed by the mode-2 log: PLL1 N divider is fed by the 10.24 MHz SYSREF SDCLKout, not by the 122.88 MHz VCXO. R1 = 24, N1 = 1, fPD1 = 10.24 MHz in NestedZeroDelay. R1 = 8, N1 = 4, fPD1 = 30.72 MHz in DualLoopMode.
- **ADR-0011 §1.2 common-mode treatment of PLL1 is broken.** PLL1 / fPD1 is not common-mode between BPMS-0.5 FPGA (mode 2, fPD1 = 10.24 MHz) and the other three configurations (mode 0, fPD1 = 30.72 MHz).
- **PLL1 floor at VCXO** (PN1Hz_PLL1 + 20·log10(N1) + 10·log10(fPD1), CP = 1550 µA): mode 0 ≈ −136 dBc/Hz; mode 2 ≈ −153 dBc/Hz (mode 2 is ~17 dB *lower* — the N1 = 1 dominates). Propagated to ADC clock (×32 = +30.1 dB): mode 0 ≈ −106 dBc/Hz; mode 2 ≈ −123 dBc/Hz.
- **In mode 0 configurations** (BPMS-0.5 ASIC, both BPMS 1.0 sat modes per ADR-0010), PLL1 at ≈ −106 dBc/Hz at the ADC clock is **worse than the PLL2 floor** computed in ADR §4. ADR-0010 and ADR-0011 §4 may be missing the dominant in-band term.

### NestedZeroDelay purpose and tradeoffs

- Purpose: deterministic phase relationship from system reference clock end-to-end out to an SDCLKout (typically SYSREF). The PLL1 loop closes around the output divider chain instead of around the VCXO, so any divider delay is inside the loop and corrected. Standard architecture for LMK04821 + JESD204B subclass-1 deterministic latency.
- Tradeoffs vs DualLoopMode: lower fPD1 (10.24 vs 30.72 MHz), narrower PLL1 loop bandwidth, sat-switch perturbs PLL1 (because CLKout_DIV changes — breaks the ADR-0010 §5.3 "PLL1 stays locked across sat-switch" claim if mode 2 is adopted), reduced runtime reconfigurability.

### Whether NestedZeroDelay is mandatory

- **Not enough evidence in the project to call it MUST.** BPMS 0.5 ASIC-sat runs DualLoopMode in production; BPMS 0.5 FPGA-sat runs NestedZeroDelay in production. Same hardware supports both. The choice is system-level.
- The deterministic-latency requirement that would force NestedZeroDelay (reference-to-sample phase identical across power cycles AND across SFU instances) has not been recorded in the ADR set or sourced to ARCH-001 / SFU-001.
- Alternatives that satisfy JESD204B subclass-1 without NestedZeroDelay exist: (B) DualLoopMode + one-shot SYSREF triggered by SYNC pin / register write, captured deterministically by RFdc; (C) DualLoopMode + continuous SYSREF + RFdc runtime latency calibration.

## Open questions

- Does BPMS 1.0 require reference-to-sample phase to be identical across power cycles? Across multiple SFU instances?
- Does BPMS 0.5 ASIC-sat (DualLoopMode in production) meet whatever deterministic-latency requirement BPMS has? If yes, that's proof DualLoopMode is sufficient.
- What does the RFdc subclass-1 implementation in the BPMS design actually need from SYSREF — phase-aligned to reference (forces NestedZeroDelay), or just captured deterministically (works in DualLoop)?
- What is the PLL1 charge-pump current setting in both ADR-0010 and the BittWare driver burst? PN1Hz_PLL1 spans −221.5 to −223 dBc/Hz across the CP settings (1.5 dB span).
- Loop bandwidths for PLL1, PLL2, LMX PLL are not specified in either ADR — required to know whether the flat-floor or flicker/VCO terms dominate the integrated jitter.
- Is the PLL2 R = 2 choice forced by the driver code, or is there a CLI flag to push it to R = 1? Same question for the LMX PLL_R = 4 → PLL_R = 1.

## Action items

- [ ] Ask Shlomi / Adi: what deterministic-latency property does BPMS 1.0 actually require at the RFdc tile? Same-SFU across power cycles, multi-SFU absolute phase, or just deterministic capture? This is the load-bearing system-level driver for LMK op mode.
- [ ] Confirm whether BPMS 0.5 ASIC-sat (DualLoopMode) meets the BPMS deterministic-latency requirement in production. If yes, NestedZeroDelay is not required; if no, it likely is.
- [ ] Open ADR-0010-OQ revision: §4 noise budget ignores PLL1 contribution at the ADC clock. In mode 0, PLL1 may be the dominant in-band term (≈ −106 dBc/Hz at ADC vs the PLL2 −113 to −115 dBc/Hz computed). Re-run the PLLatinum Sim task (ADR-0010-OQ-1) explicitly including PLL1 + Si5394.
- [ ] Open ADR-0011 v0.4 revision: §1.2 common-mode treatment is broken (PLL1 differs between mode 0 and mode 2 configurations). §4 may understate the in-band floor in mode-0 configurations. Validation-by-analogy argument in §6.1 still survives because BPMS-0.5 FPGA-sat (mode 2) has a *lower* PLL1 floor than ADR assumed, but the underlying accounting must be corrected.
- [ ] Decision on LMK op mode for BPMS 1.0: either (a) commit to NestedZeroDelay and revise ADR-0009 sat-switch contract (PLL1 re-acquires on every switch), or (b) commit to DualLoopMode and document the JESD204B subclass-1 approach used (SYSREF capture rather than reference-aligned SYSREF). Either way, record the decision and its system-level driver in a new or extended ADR.
- [ ] Confirm BittWare driver supports forcing LMK PLL2 R = 1 / OSC_2X = 1 and LMX PLL_R = 1; if not, evaluate bypassing `rfx_clock_setup.elf` with a TICS-Pro-generated register burst pushed via the same SPI path. ~3 dB of LMK floor is recoverable here.

## Re-seed block

GOAL: decide LMK04821 op mode and driver-vs-ADR delta resolution for BPMS 1.0 H.1 clock-tree plan, with PLL1 contribution explicitly accounted for.
CONSTRAINTS:
- Hardware fixed: RFX-8440A, build option B, 122.88 MHz Crystek CVHD-950 VCXO, Si5394 OUT3 = 245.76 MHz reference.
- LMX2594 `VCO_PHASE_SYNC = 1` required (TX-RX deterministic phase between ADC-LMX and DAC-LMX) — confirmed non-negotiable in earlier session. Forces IncludedDivide = 4, fPFD_LMX ≤ ~65 MHz at BPMS sample rates.
- ADC sample rates: 4096 MS/s (ASIC-sat) / 4194.304 MS/s (FPGA-sat) for BPMS 1.0. LMX VCO1, CHDIV = 2 in both modes.
- Integer-N at LMX2594 (no SDM noise / integer-boundary spurs).
PRIOR CONCLUSIONS:
- Driver-vs-ADR flat-floor delta is consistently ≈ +3 dB across all three configurations examined (BPMS 1.0 ASIC, BPMS 1.0 FPGA, BPMS 0.5 FPGA mode 2). Decomposes into +3 dB on LMK PLL2 (driver R=2 / N_total doubled vs ADR R=1) and +6 dB on LMX self-noise (VCO_PHASE_SYNC IncludedDivide=4), with LMK dominating the total.
- BittWare PLL1 finding confirmed: NestedZeroDelay uses R1=24, N1=1, fPD1=10.24 MHz (feedback from SYSREF SDCLKout). DualLoopMode uses R1=8, N1=4, fPD1=30.72 MHz (feedback from VCXO directly).
- PLL1 floor at ADC clock: ≈ −106 dBc/Hz in mode 0, ≈ −123 dBc/Hz in mode 2. In mode 0 configurations, PLL1 likely dominates the in-band budget, not PLL2 — both ADR-0010 and ADR-0011 §4 currently miss this.
- NestedZeroDelay vs DualLoopMode is not a phase-noise feature but a deterministic-latency feature. It satisfies JESD204B subclass-1 with reference-aligned SYSREF. DualLoopMode satisfies subclass-1 with captured-SYSREF approach. Both are supported by Xilinx RFdc.
- BPMS 0.5 ASIC-sat runs DualLoopMode in production; BPMS 0.5 FPGA-sat runs NestedZeroDelay in production. Same hardware supports both. The choice is system-architectural, not hardware-forced.
CURRENT QUESTION: what is the BPMS 1.0 deterministic-latency requirement at the RFdc interface — sufficient to force NestedZeroDelay, or compatible with DualLoopMode + captured-SYSREF? This drives the ADR-0010 § sat-switch contract (which assumes PLL1 stays locked across sat-switch — only valid in DualLoopMode) and the ADR-0011 revision (which assumed PLL1 common-mode — broken if BPMS-0.5 FPGA-sat uses NestedZeroDelay and others use DualLoopMode).
