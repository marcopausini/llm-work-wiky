# Review request

**Artefact under review:** `arch/fad/FAD.md` §4 Clocking and Reset Architecture at commit `<hash>` | version `<FAD version>` | paste-in below
**Target gate:** G1 Orient
**Scope:** focused review of §4 only — clock inventory, clock topology, primary/secondary clock modes, reset topology, CDC inventory

**Context for this review:**
- FAD §1, §2, and §6 are the current baseline context for this review.
- FAD §3 is intentionally collapsed; dataflow narrative is incorporated into §2.
- FAD §4 is being reviewed as the clock/reset/CDC section that feeds:
  - every future Module Spec “Clocks and resets” table
  - the CDC inventory used by execution-flow CDC checks
  - ADR-0002 clock architecture
  - top-level constraints planning
- Core ICDs (`streaming_bus`, `register_bus`, `obg_frame`) are still draft / not frozen.
- Module Specs have not started; do not require MS-level completeness yet.
- ADR-0002 clock architecture is pending and should be triggered by any non-trivial clocking decision.
- Known open items already tracked — do not re-flag unless §4 contradicts them:
  - exact Aurora IP user-clock naming/rates may be TBD until IP configuration is created
  - exact MMCM/PLL primitive choice may be TBD unless already sourced
  - exact CDC FIFO depth is uArch scope, not FAD scope
  - `obg_link` owns Aurora ↔ DSP CDC mechanism at contract level; FIFO sizing is designer uArch scope
  - `utc_apply_bus` ICD / ADR-0004 is pending before first UTC-scheduled-parameter MS
  - PS/PL partitioning details for TLE/SGP4 and management protocol are pending ADR-0003

**Source pins:**
- BPMS-1.0-SFU-001 v1.6:
  - §6.1 Aurora / OBG Input and DL CDC
  - §6.2 OBG Multi-Lane Phase Alignment
  - §6.4 Filter Bank Synthesis
  - §6.6 Band Doppler DL
  - §7.1–§7.5 UL chain
  - §9.1 RFX-8440A FPGA Card
  - §11 Timing and Synchronisation
  - §12.2 RF and ADC/DAC Parameters
  - §17 SFU Management
- BPMS-1.0-ARCH-001 v2.4:
  - §5.2 Signal Chain
  - §5.6 Band-Level vs Beam-Level Processing
  - §11 Synchronisation & Clock Distribution
  - §12.1 SFU External Interfaces
  - §15.2 UTC-Scheduled Parameter Apply Mechanism
  - §15.3 Parameter Scope — Scheduled Apply vs Local Autonomous
  - §22 Consolidated Requirements
- FAD context:
  - §1.2 Functional boundary
  - §1.4 Key architectural decisions
  - §2 Top-Level Block Diagram and Dataflow
  - §6 Module Inventory
  - §13 Open Items
- Methodology:
  - `docs/methodology/01-fpga-project-methodology.md`
  - `docs/methodology/02-architect-workflow.md`
  - `docs/methodology/04-execution-flow.md`
  - `docs/methodology/05-signoff-criteria.md`
- ADRs:
  - ADR-0001 5-doc framework
  - ADR-0005 Module Spec / uArch split, if accepted in repo; otherwise treat as pending and flag only if §4 depends on its acceptance
  - ADR-0002 pending — clock architecture

**Special asks:**
- Verify §4.1 clock inventory covers every clock domain implied by FAD §2 and §6:
  - Aurora / SERDES recovered/user clock domain
  - SFU DSP processing domain
  - filter-bank input/output clock domains, including any 341.333 MHz / 320 MHz references
  - RFDC converter / DUC / DDC related clocks
  - register / management clock domain
  - 1PPS / UTC timing capture domain
  - non-volatile configuration / boot-restore domain, if referenced
  - debug/capture/playback clock domains, if referenced
- Verify primary clock mode is consistent with SFU-001 §11.1 / §6.1:
  - operational clock recovered from a selected OBG DL Aurora lane using CDR
  - no shared clock between SFU cards unless explicitly stated
  - recovered clock relationship to FPGA logic and RFDC clocks is stated without inventing unsupported implementation details
- Verify secondary clock mode is consistent with SFU-001 §11.2 and ARCH-001 §11:
  - external 10 MHz mode is described clearly
  - clock-source selection / switchover behavior is marked TBD or specified with source
  - no unsafe claim of hitless switchover unless sourced
- Verify 1PPS treatment:
  - 1PPS is a timing event / apply trigger, not a high-speed data clock
  - 1PPS crossing into PL clock domain is explicitly captured
  - Band Doppler Local Autonomous 1PPS apply is separated from UTC-scheduled parameter apply
- Verify reset topology:
  - reset assertion/deassertion style is defined per clock domain
  - reset release is gated by relevant lock/ready conditions where required, or marked TBD
  - reset-domain crossings are considered, not only data CDCs
  - mode changes and clock-source changes have safe reset/reinitialization behavior or explicit TBD markers
- Verify CDC inventory §4.5:
  - every intentional CDC has an ID, source domain, destination domain, data/control type, mechanism, and owning MS/block
  - mechanisms are approved: async FIFO, two-flop synchronizer, handshake, vendor CDC/IP wrapper, or explicitly justified TBD
  - no ad-hoc multi-bit synchronizer is implied
  - Aurora ↔ DSP CDC is assigned to `obg_link`
  - low-rate CSR/config crossings are covered
  - 1PPS / UTC / event-log crossings are covered
  - RFDC / DSP boundary is covered if clock domains differ
  - filter-bank internal `clk_in` / `clk_out` crossing is covered or explicitly delegated to the vendor/IP boundary
- Check that §4 does not leak uArch:
  - do not require FIFO depth, exact synchronizer implementation, FSMs, pipeline depth, or internal RTL decomposition
  - FAD should specify the required clock relationship and approved CDC mechanism, not implementation internals
- Check citation discipline:
  - every factual source-derived claim should cite SFU-001, ARCH-001, FAD, ICD, or ADR section
  - every inferred/proposed item should be marked `[INFERRED]`, `[ASSUMPTION]`, `[TBD]`, or `[STUB]`
- Check consistency with downstream gates:
  - `constraints/00_clocks.xdc` must be able to derive clocks from §4.1 later
  - `04_clock_groups.xdc` must be able to derive async/sync relationships from §4
  - `report_cdc` results must be reconcilable with §4.5
- Output only review findings. Do not rewrite §4.

**Required output format:**

| Severity | Location | Issue | Why it matters | Recommended action | Gate impact |
|---|---|---|---|---|---|
| blocker / major / minor / question | §4.x / table row / paragraph | concise issue | concrete impact | specific fix | G1 / later G2-G4 / none |

After the table, add:

## Gate assessment

- **G1 status:** pass / pass with minors / blocked
- **Blocking items:** numbered list, or “none”
- **ADR triggers:** list any required ADR-0002 / ADR-0003 / ADR-0004 follow-ups
- **Do-not-change:** list any parts that are correct and should be preserved

---

[paste FAD §4 content here]