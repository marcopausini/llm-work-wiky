---
id: 2026-05-08--bpms-sfu-architecture--sfu-clock-topology-review
date: 2026-05-08
project: bpms-sfu-architecture
type: insight
status: open
topics: [clocking, rfx-8440a, si5394, lmk04821, lmx2594, rfdc, cdc, phase-coherence, fad-section-4, solution-b, holdover]
source_chat: claude-sonnet-4-6
---

# FAD §4 clock-topology inversion found; Solution B grounded in RFX-8440A board topology

## Context

Review of FAD §4 v0.5 (clocking and reset architecture) triggered by a reviewer
blocker. §4 v0.5 was produced in the previous session integrating ADR-0006 and
ADR-0007. Reviewer challenged §4.2.1 (clock chain framing) and §4.5.2
(synchronous-derived claim). Session reconstructed the architecture from
SFU-001 v1.6 and the RFX-8440A clock-tree diagram, teaching the fundamentals
along the way. Adi (FPGA Design Lead) requested a clear architectural
clock diagram for the next meeting.

## Key findings

- **FAD §4 v0.5 inversion.** §4.2.1 places the LMK as the master clock source
  with the OBG-recovered clock as a "discipline reference." SFU-001 v1.6 §11.1
  states the opposite: "recovered clock drives all SFU FPGA logic and ADC/DAC
  converters." The parent doc framing puts the recovered clock at the top of the chain.

- **Why clock-from-serial-lane.** Embeds clock in data stream. No separate clock
  distribution network. Polatis L1 matrix routes data+clock together automatically.
  Scales without adding infrastructure per SFU. Satisfies inter-SFU frequency-lock
  for free.

- **Why an LMK chain is still required.** Two clocks must enter the FPGA from
  outside on dedicated pins: (1) RFdc tile reference (ADC/DAC sample clocks, separate
  clock island, sub-100 fs jitter required); (2) GT TX refclk (Aurora SERDES TX,
  dedicated GTREFCLK pin). FPGA-internal MMCMs cannot generate either. The
  LMK chain bridges the recovered clock to these two dedicated inputs.

- **Actual board topology (RFX-8440A clock-tree diagram confirmed).** The
  "LMK/LMX" abstraction is a three-stage chain:
  - Si5394 (jitter cleaner + clock generator) — takes OBG-recovered clock as
    primary input; has 48 MHz crystal at XAB for digital PLL local oscillator
    and digital holdover
  - LMK04821 (PLL + clock distribution) — fed from Si5394 O3 via build-option
    mux; has programmable VCXO for loop and holdover
  - LMX2594 ×2 (RF-class PLLs) — fed from LMK04821; output RFDC sample-clock
    references directly

- **GT TX refclk and RFDC ref are structurally separate.** Si5394 O0/O1 drive
  MGT TX refclks. Si5394 O3 → LMK04821 → LMX2594 drives RFDC refs. These two
  paths share the Si5394 but diverge after it. Sample-rate switching (reprogramming
  LMK04821 + LMX2594) does NOT touch Si5394 O0/O1. The "RFDC reprogramming
  disturbs GT TX refclk" concern is resolved structurally by board topology —
  sub-question of OQ-§4-4 is a non-issue.

- **Holdover is two-stage.** Si5394 holds last frequency via 48 MHz crystal
  digital memory. LMK04821 holds via VCXO. Together they provide robust holdover
  when OBG drops. Holdover budget (duration / phase-error) inherits from
  ARCH-001 v2.4 OI-013 — still TBD at system level.

- **OBG drops are real operational events.** Causes: Polatis matrix
  reconfiguration on satellite handover, DCU reset, CPRI upstream failure, fibre
  maintenance, bring-up/commissioning. Not frequent but happen at every
  maintenance window and every satellite-pass matrix reconfigure.

- **Serial-lane clock gives frequency coherence, NOT phase coherence.**
  All SFUs in a G-SFU group derive clocks from the same DCU source via OBG →
  frequency lock is automatic. Phase offset between SFUs is deterministic
  (set by differential fibre lengths through Polatis matrix, GT CDR static
  offset, on-chip clock-tree delay) but NOT zero. SFU-001 v1.6 §11.5 / 
  ARCH-001 v2.4 §11.5 claim phase coherence is "inherent" — this is true at the
  frequency level only, not the phase level. A per-SFU phase-trim register +
  commissioning calibration is the second required step.

- **clk_dsp ↔ clk_rfdc_axis "synchronous-derived 1:3" remains [ASSUMPTION].**
  The claim depends on the MMCM having a fixed M/D ratio from clk_rfdc_axis. This
  is architecturally intended but depends on OQ-§4-3 (MMCM VCO valid at both rates
  × chosen M) and the overall §4.2.1 topology being confirmed.

- **Solution A vs Solution B are the same hardware at different tuning points.**
  A: board crystal/OCXO as steady-state master, OBG-recovered as slow discipline.
  B: OBG-recovered as master, OCXO as holdover backup. A is more robust to
  transients (LMK barely notices OBG drop); B is architecturally cleaner and
  aligns literally with SFU-001 §11.1. Neither is ruled out. The interpretation
  of "drives" in SFU-001 §11.1 determines which is normative.

- **Secondary mode topology.** RF Ext Ref In (CI port, 10 MHz) goes directly
  to LMK04821, bypassing Si5394 entirely. In secondary mode, what drives Si5394
  (and thus MGT TX refclks) is an open question — likely Si5394 runs in digital
  holdover on 48 MHz crystal.

- **Build-option mux at LMK04821 input is critical.** The path Si5394 O3 →
  LMK04821 is configured at board manufacture. If the RFX-8440A build selects
  Fixed 10 MHz instead, Solution B topology does not hold. Needs HW Lead
  confirmation.

## Decisions

- FAD §4.2.1 must be rewritten to put the OBG-recovered clock at the top of
  the chain, with the Si5394 as a slave jitter cleaner, not the master source.
- The three-stage chain (Si5394 → LMK04821 → LMX2594) replaces the generic
  "LMK/LMX" abstraction throughout §4.
- §4.5.2 "synchronous-derived 1:3" row for clk_dsp ↔ clk_rfdc_axis must be
  marked `[ASSUMPTION]` pending ADR-0002 acceptance and OQ-§4-3 confirmation.
- FAD-DEC-17 ("synchronous-derived 1:3 via MMCM") downgraded from confirmed
  to `proposed`.
- Per-SFU phase-trim register is architecturally required (pending OI-S02);
  SFU-001 §11.5 "inherent" wording is misleading and should be flagged to Shlomi.

## Open questions

- **OQ-§4-4 (sharper sub-questions for HW Lead):**
  1. Is MGT128 RX recovered clock physically routed to a Si5394 input on the
     standard RFX-8440A build?
  2. Is the build-time mux at LMK04821 input configured to select Si5394 O3
     (not Fixed 10 MHz, not pl_ref_clk)?
  3. Is the VCXO the programmable variant (OscFrqCfg = C) — required to support
     both 4096 and 4194.304 MSPS?
  4. In secondary clock mode (RF Ext Ref In → LMK04821 direct), what drives the
     Si5394? Do MGT TX refclks stay valid?
- **OQ-§4-10:** Is SFU-001 §11.1 "recovered clock drives ADC/DAC converters"
  intended literally (bit-derived clock) or loosely (GPS-traceable via recovered
  clock)? Owner: Shlomi.
- **OQ-§4-11:** GPS holdover duration and accumulated phase-error budget at SFU
  level. Inherits ARCH-001 v2.4 OI-013. Owner: Sys Arch.
- **OQ-§4-3:** MMCM VCO range valid at both clk_rfdc_axis values × chosen M/D.
  Owner: RTL Designer. Expiry: first OOC synthesis.
- **OI-S02 / SYS-CLK-09:** Per-SFU phase-trim register: needed? how
  specified? Owner: Sys Arch. Expiry: obg_link MS authoring.

## Action items

- [ ] Produce the SFU clocking architectural diagram for Adi's review (next chat)
- [ ] Rewrite FAD §4.2.1: OBG-recovered clock at top of chain; Si5394 as
      jitter cleaner; LMK04821 + LMX2594 as downstream slave chain; board-topology
      details marked `[ASSUMPTION: confirmed by HW Lead OQ-§4-4]`
- [ ] Soften FAD §4.0 mechanism statement ("DynamicPLLConfig + MMCM cascade")
      with `[ASSUMPTION]` pending OQ-§4-3
- [ ] Add `[ASSUMPTION]` to §4.5.2 "synchronous-derived 1:3" row for
      clk_dsp ↔ clk_rfdc_axis
- [ ] Downgrade FAD-DEC-17 to `proposed` in FAD §1.4 decisions list
- [ ] Replace generic OQ-§4-4 with four sharper sub-questions above
- [ ] Add OQ-§4-11 (holdover budget) to FAD §4.8
- [ ] Flag SFU-001 §11.5 "phase coherence is inherent" wording to Shlomi —
      this is frequency coherence only; phase trim is the missing second step
- [ ] Confirm Si5394 three-stage chain naming in FAD §4.2.1 (not "LMK/LMX")
- [ ] Confirm VCXO option C in the BittWare order (if not already)

## Re-seed block

GOAL: Produce a clear architectural diagram of the SFU FPGA clocking system
for Adi (FPGA Design Lead) review.

CONSTRAINTS:
- Target device XCZU43DR-2FFVE1156E on BittWare RFX-8440A
- Board clock chain confirmed: Si5394 (jitter cleaner) → LMK04821 (PLL
  distribution) → LMX2594 ×2 (RF PLLs for ADC/DAC RFDC tile references)
- Si5394 O0/O1 → MGT TX refclks (sample-rate-invariant)
- Si5394 O3 → LMK04821 via build-option mux (needs HW Lead confirmation)
- Four FPGA clock domains: clk_aurora_user / clk_rfdc_axis / clk_dsp / clk_mgmt
- clk_dsp ↔ clk_rfdc_axis synchronous-derived 1:3 via MMCM (ASSUMPTION, not yet confirmed)
- clk_aurora_user ↔ clk_dsp is the only fundamentally async datapath crossing
- OBG-recovered clock is the master timing source (Solution B); Si5394 is the
  entry point; OBG drop → LMK enters holdover via VCXO / Si5394 crystal

PRIOR CONCLUSIONS:
- FAD §4 v0.5 had LMK as master — this is inverted from SFU-001 §11.1; §4.2.1
  needs rewrite before FAD.md merge
- Serial-lane clock recovery gives frequency coherence between SFUs for free;
  phase coherence requires separate per-SFU phase-trim + commissioning calibration
- Sample-rate switching only reprograms LMK04821 + LMX2594; Si5394 MGT outputs
  are untouched; Aurora link preserved by board topology
- "synchronous-derived 1:3" claim for clk_dsp ↔ clk_rfdc_axis is still [ASSUMPTION]
- Four key HW Lead questions needed before §4.2.1 can be marked confirmed

CURRENT QUESTION: Draw the SFU clocking diagram from the OBG input through the
Si5394 → LMK04821 → LMX2594 chain to all four FPGA clock domains, showing
holdover paths, sample-rate-scaling annotations, and the four boundary OQs.
Diagram must be clear enough for Adi to review at a team meeting.
