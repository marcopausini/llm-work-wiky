---
doc_id: BPMS-SFU-METH-005
doc_type: methodology
project: bpms-sfu-fpga-design
status: draft
version: 0.2
date: 2026-04-27
author: Marco Pausini
---

# FPGA Sign-off Criteria

This document defines what *passing* means at every gate in the BPMS 1.0 SFU FPGA project — from document sign-off through hardware bring-up. It is the *what counts as done* counterpart to [04-execution-flow.md](04-execution-flow.md) (which defines *how to run the build*). Every section below opens with a pointer to the matching gate command in the flow document.

Parent document: [01-fpga-project-methodology.md](01-fpga-project-methodology.md).

---

## 1. Purpose and scope

In scope:

- Document sign-off (FAD, ICD, MDS / module spec, RTM, ADR)
- Per-stage flow sign-off (lint, unit sim, OOC synth, CDC, top synth, implementation, constraints)
- Architecture review gates (G1 Orient, G2 Contracts, G3 Budgets)
- Module gates G4a (Spec, architect) and G4b (uArch / RTL, designer)
- Integration gate (per module)
- DV handoff (G5)
- Hardware bring-up sign-off
- Release candidate checklist

Out of scope (covered elsewhere):

- Gate commands and report locations → [04-execution-flow.md](04-execution-flow.md)
- XDC structure and exception fields → [../../constraints/README.md](../../constraints/README.md)
- Waiver discipline → [../../waivers/README.md](../../waivers/README.md)
- LLM-assisted usage policy → [04-execution-flow.md](04-execution-flow.md) §16 (Claude usage is consolidated there per the framework split)

---

## 2. Sign-off levels overview

```
                 G1   G2   G3                       G4a               G4b   integ.    G5         HW
Architecture  ───●────●────●─────────────────────────●─────────────────────────────────────────────
Spec / refmodel                                      ●(per module)
RTL lint                                                              ●(per module)
Unit sim                                                              ●(per module)
OOC synth                                                             ●(per module)
CDC                                                                   ●(per module) ●(top)
Constraints                                                                          ●(top)
Top synth                                                                            ●(top)
Implementation                                                                       ●(top)
DV / UVM                                                                                       ●
Hardware bring-up                                                                                       ●
```

Each sign-off must be green at its own commit. Promotion to a downstream gate (e.g. integration, DV handoff) requires every upstream gate at the same or earlier commit to be green at that commit.

No gate may be waived silently. A waiver is recorded as an ADR (architecture-level) or a `waivers/` entry (tool-level) and reviewed at the gate it affects.

---

## 3. Documentation sign-off

**Gate command:** see [04-execution-flow.md](04-execution-flow.md) §4 (`make doc_check`).

Documentation sign-off applies to FAD, ICDs, MDS / module specs, ADRs, and the RTM. Lifecycle states are defined in [../../arch/README.md](../../arch/README.md). The criteria below are what must be true to advance to the next state.

### 3.1 FAD sign-off (per gate G1, G2, G3)

A FAD section reaches `baselined` when:

- All factual claims carry an inline citation OR a `[INFERRED]` / `[ASSUMPTION]` marker
- Every placeholder marker has the required fields (TBD owner, STUB blocker, ASSUMPTION expiry trigger)
- Module inventory (§6) and top-level diagram (§2) are consistent — every block in §2 has a row in §6 and a matching `arch/modules/<m>.md`
- Architecture invariants (§1.5) are stated; no other section violates them
- Clock inventory (§4.1) and CDC inventory (§4.5) cover every domain and every crossing referenced anywhere in the project

### 3.2 ICD sign-off

An ICD reaches `frozen` when:

- Signal list, payload format, and sideband format are complete
- Timing rules (handshake, backpressure, framing) are unambiguous, with at least one timing diagram or waveform
- Protocol assertions (§6 of the ICD template, IF-SVA-NNN rows) are populated
- Every consumer module is listed in the frontmatter and §1.3
- Versioning policy is stated (§7)

After `frozen`, additive changes bump minor version; backwards-incompatible changes bump major version and require an ADR.

### 3.3 Module spec sign-off (gate G4a — see §12.1)

Defined in §12.1 below.

### 3.4 RTM sign-off

The RTM is `living` and never `baselined`. Spot-checks at every gate:

- Every requirement has at least one row
- Every row maps to a module (or carries `[STUB]` with owner and expiry trigger)
- Every closed row has evidence (test report, coverage extract, review minutes)
- Derived requirements use the `FAD-DER-<CATEGORY>-NNN` ID format and cite the parent that forced them
- Status counts in the summary table match the row statuses

### 3.5 ADR sign-off

An ADR reaches `accepted` when:

- Context, decision, alternatives considered, and consequences are present
- Every alternative has at least one Pro and one Con
- The decision cites the forces that justify it (constraints, parent requirements, prior ADRs)
- Follow-ups required are listed and either tracked as tasks or as derived requirements

ADRs are immutable after `accepted` — superseding requires a new ADR that links the predecessor.

---

## 4. RTL lint sign-off

**Gate command:** see [04-execution-flow.md](04-execution-flow.md) §5 (`make lint MOD=<m>`).

A module passes lint sign-off when:

- Zero `error` and zero `fatal` rule violations
- Every `warning` is either fixed or covered by a `waivers/lint/<m>.waiver` entry with required fields per [../../waivers/README.md](../../waivers/README.md)
- Naming conventions enforced by the lint rule set are clean
- Protocol-level SVA rules (from referenced ICDs §6) compile without warnings

A waived warning blocks lint sign-off if its waiver lacks an expiry trigger.

---

## 5. Unit simulation sign-off

**Gate command:** see [04-execution-flow.md](04-execution-flow.md) §6 (`make sim_unit MOD=<m>`).

A module passes unit simulation sign-off when:

- Every directed scenario in MDS §11.1 passes
- Constrained-random tests run for the duration specified in the unit-TB plan with zero unexpected failures
- Bit-exactness criterion (per MDS §3.3 and §7.3) is met:
  - `bit_exact` → zero LSB difference at every sample on the measurement boundary
  - `ulp_bounded (bound=N)` → all samples within ±N ULP at the measurement boundary
  - `not_applicable` → boundary metric (spectral mask, EVM, etc.) per MDS §7.3 is met
- Coverage targets in MDS §11.3 are met or exceeded:
  - 100% FSM states and transitions
  - ≥ 95% line coverage
  - ≥ 95% toggle coverage
  - 100% branch coverage on protocol paths
  - All functional bins listed in MDS §11.3 hit
- All SVA assertions hit at least once; no false-positive firings
- Reset-mid-operation scenario passes for every module

---

## 6. Out-of-context synthesis sign-off

**Gate command:** see [04-execution-flow.md](04-execution-flow.md) §7 (`make synth_ooc MOD=<m>`).

A module passes OOC synthesis sign-off when:

- WNS ≥ 0 ns and TNS = 0 ns at the OOC clock period stated in `constraints/ooc/<m>.xdc`
- Setup and hold both clean (no negative slack on either)
- Resource utilisation within +20% of the MDS §12.1 estimate; if exceeded, MDS §12.2 / §13 records the deviation with rationale
- No critical warnings about inferred latches, multi-driven nets, or missing constraints
- Per-clock check_timing report shows no unconstrained paths
- DSP / BRAM / URAM inference matches the uarch's stated inference plan (no DSP slices missed where the uarch said DSP would be inferred)

---

## 7. CDC sign-off

**Gate command:** see [04-execution-flow.md](04-execution-flow.md) §11 (`make cdc MOD=<m>` per module, `make cdc` at top).

CDC sign-off (per module and at top level) requires:

- Every reported CDC violation is either:
  - Resolved with an approved CDC mechanism per FAD §4.5 (two-flop synchroniser, async FIFO, handshake, vendor IP), or
  - Covered by a `waivers/cdc/` entry with a documented analysis (timing-quasi-static argument, custom synchroniser proof) and an expiry trigger
- The CDC report's crossing list is bidirectionally consistent with FAD §4.5: every report row has an inventory entry; every inventory entry appears in the report
- No multi-bit two-flop synchroniser without a quasi-static contract
- Reset-domain crossings analysed alongside data-domain crossings (per UG949)
- The third-party CDC tool (when adopted) agrees with `report_cdc` on every flagged crossing

---

## 8. Top-level synthesis sign-off

**Gate command:** see [04-execution-flow.md](04-execution-flow.md) §8 (`make synth_top`).

Top-level synthesis sign-off requires:

- Synthesis completes without errors; warnings reviewed and either fixed or noted in the run log
- Estimated WNS at synthesis ≥ 0 ns under the primary clock set
- Resource utilisation within FAD §9.2 budget; hard-fail above 85% utilisation on any single resource type
- Every clock in the design appears in the `report_clocks` output and matches FAD §4.1
- DRC clean at the synthesis level
- IP output products are present and from the pinned Vivado version (§2 of execution-flow)

---

## 9. Implementation sign-off

**Gate command:** see [04-execution-flow.md](04-execution-flow.md) §9 (`make impl`, `make timing`, `make cdc`).

Implementation sign-off requires:

- WNS ≥ 0 ns and TNS = 0 ns post-route, on every clock and every clock-pair (intra and inter)
- Hold timing clean (no negative slack)
- `report_drc` clean at sign-off severity (no errors; warnings reviewed)
- `check_timing` clean: no unconstrained paths, no overridden constraints, no multiple-driver constraints
- `report_cdc` clean per §7 above (top-level)
- Power within FAD §9.4 budget; if exceeded, ADR or waiver
- Resource utilisation within FAD §9.2 budget on every resource type
- Bitstream generated; CRC and identification fields verified

---

## 10. Constraints sign-off

**Gate command:** see [04-execution-flow.md](04-execution-flow.md) §10 (the constraints methodology section).

Constraints sign-off requires:

- Every clock in `constraints/00_clocks.xdc` matches FAD §4.1
- Every generated clock in `01_generated_clocks.xdc` is justified (PLL/MMCM ratio, divider, etc.)
- `04_clock_groups.xdc` declares every asynchronous clock relationship — no missing async groups
- Every entry in `05_timing_exceptions.xdc` has the required fields per [../../constraints/README.md](../../constraints/README.md): reason, owner, source/destination objects, proof reference, review status
- Every timing exception either has a `waivers/timing/` entry with expiry, or is structurally permanent (e.g. mode-mux false path) with proof in MDS / ADR
- IP-generated XDC loads before project XDC; no manual edits to IP-generated files
- `check_timing` clean (no overridden, no missing, no multiple-driver)

---

## 11. Architecture review gates (G1, G2, G3)

These are architect-owned gates that promote FAD sections to `baselined`. Detailed architect-side checks are in [02-architect-workflow.md](02-architect-workflow.md) §7.

### 11.1 G1 Orient

**Trigger:** FAD §1–§6 complete.

**Pass criteria:**

- Functional boundary table (§1.2) populated; ownership clear for every function in scope and out of scope
- Architecture invariants (§1.5) stated and not violated by §2–§6
- Top-level block diagram (§2) and module inventory (§6) consistent (every block has a row; every row points to a module spec)
- Dataflow narrative (§3) covers DL, UL, and debug/auxiliary paths
- Clock inventory (§4.1) covers all domains; CDC inventory (§4.5) covers all crossings referenced in §3
- Memory architecture (§5) lists all on-chip memory by function

**Evidence:** reviewed FAD §1–§6 committed; review issue table resolved (architect-owned, archived per [02-architect-workflow.md](02-architect-workflow.md) §6.6).

### 11.2 G2 Contracts

**Trigger:** FAD §7–§8 complete; core ICDs ready to freeze.

**Pass criteria:**

- Streaming-bus convention (§7.1) unambiguous; every parameter has a default
- Register-bus convention (§7.2) unambiguous; CDC rules stated
- Backpressure rule (§7.3) covers stall, drop, and frame-boundary handling
- Fixed-point policy (§8) covers every stage boundary in §3 dataflow
- Core ICDs (streaming_bus, register_bus, obg_frame) reach `frozen` (§3.2 above)
- All consumers listed in each ICD frontmatter

**Evidence:** reviewed FAD §7–§8 + ICDs committed; ICDs at `status: frozen`.

### 11.3 G3 Budgets

**Trigger:** FAD §9–§12 complete.

**Pass criteria:**

- Latency budget (§9.1): every pipeline stage has a number or `[TBD]` with owner and expiry trigger
- Resource budget (§9.2): every module has a number or `[TBD]` with owner; total within device capacity with margin
- Floorplan intent (§9.3): SLR topology, RF-facing and transceiver-facing placement documented
- Power budget (§9.4): preliminary XPE estimate or `[TBD]` with owner
- Debug architecture (§10): minimum telemetry set defined; capture/playback points identified
- Management plane (§11): register bus topology, scheduled apply, autonomous update documented
- Verification contract (§12): two-tier model defined; verification matrix populated

**Evidence:** reviewed FAD §9–§12 committed.

---

## 12. Module gates: G4a (Spec) and G4b (uArch / RTL)

The module-level gate is split per the role boundary. The architect signs off the external contract; the designer signs off the internal implementation and its evidence. The split (per the project's spec/uarch terminology) is described in [01-fpga-project-methodology.md](01-fpga-project-methodology.md) §3.1 and [02-architect-workflow.md](02-architect-workflow.md) §7.4–§7.5.

### 12.1 G4a — Spec sign-off (architect)

**Owner:** FPGA architect.

**Trigger:** module spec drafted; reference model (for DSP/numerical modules) authored and passes spec tests.

**Pass criteria** (`status: design_ready` per ADR-0005 — see [../../arch/README.md](../../arch/README.md) and [01-fpga-project-methodology.md](01-fpga-project-methodology.md) §3.1 / §7.4; the legacy state `rtl_ready` is retired):

- All interfaces resolved against frozen ICDs (template §5.1, §5.3)
- Clocks, resets, CDC specified (template §6); every CDC mirrored in FAD §4.5
- All parameters carry type, default, legal range, and effect (template §5.2)
- Fixed-point format specified at every interface and significant internal node (template §7.1)
- Algorithm or FSM fully specified, OR refmodel pointer + bit-exactness policy set (template §3.3, §7.3)
- Latency, throughput, backpressure specified (template §8.1)
- Register map: minimum register set defined (template §5.3) — high-level; address/bit assignment is designer scope
- Error and corner-case behaviour specified (template §10)
- Reference model exists and passes its spec tests (template §3.3, for DSP/numerical modules)
- Verification hooks enumerated (template §11.4)
- Traceability to SYS-* / SFU-* / FAD-DER-* requirements complete (template §1)
- No internal micro-architecture prescribed (no FSM internals, pipeline depth, FIFO impl, sub-block decomposition)
- Architecture invariants (FAD §1.5) respected

**Evidence:** module spec committed at the appropriate `status` value with empty blocking list; review issue table resolved.

### 12.2 G4b — uArch / RTL sign-off (RTL designer)

**Owner:** RTL designer.

**Trigger:** module spec at G4a-pass status; designer has authored the micro-architecture and RTL.

**Pass criteria:**

- Micro-architecture document (`uarch.md`, designer's location of choice) authored, covering: pipeline stages, FSM internals, FIFO depths, sub-block decomposition, internal Q-formats, DSP/BRAM/URAM inference plan
- RTL implements the spec without surprising the spec reader: every spec'd port, parameter, register, mode, and external behaviour is observable on the RTL boundary
- Lint sign-off green (§4)
- Unit simulation sign-off green (§5), including bit-exactness per MDS §3.3
- OOC synthesis sign-off green (§6)
- CDC sign-off green at module level (§7)
- MDS §12.3 evidence table populated for the lint / unit sim / OOC / CDC rows; commit hashes recorded
- All TODOs / open items in the uarch closed or escalated to MDS §13
- RTM rows for the module updated (`in_design` → `in_rtl`)

**Architect's review at G4b** (consistency check, not a re-implementation review):

- Implementation respects the spec contract (no undocumented external behaviour)
- CDC mechanisms match those listed in the spec and FAD §4.5
- Resource usage within budgeted ceiling (FAD §9.2)
- No undocumented functional boundary crossing (FAD §1.2)

**Evidence:** RTL committed; uarch committed; reports archived; MDS §12.3 populated; architect consistency review noted in the PR / review record.

---

## 13. Integration gate (per module)

**Gate command:** triggered by populating MDS §12.3 — see [04-execution-flow.md](04-execution-flow.md) §3 (`make reports MOD=<m>`).

A module is **integration-ready** (eligible for inclusion in `make synth_top`) when:

- G4a passed (spec)
- G4b passed (uarch + RTL)
- MDS §12.3 evidence table fully populated with status = pass for: lint, unit_sim, coverage, synth_ooc, timing_summary, cdc, check_timing
- Module-level XDC (`rtl/<m>/<m>.xdc`) reviewed and merged
- OOC constraints (`constraints/ooc/<m>.xdc`) reviewed and merged
- All waivers referenced from this module's reports have entries in `waivers/`
- Frontmatter `implementation_evidence: complete` (when this field is adopted; see [../../arch/README.md](../../arch/README.md))

The integration gate is workflow state, not a document lifecycle state — per the framework split (Change 6 in the revision plan). Document `status` and implementation evidence are tracked separately.

---

## 14. DV handoff (G5)

**Owner:** Verification engineer (in `bpms-sfu-fpga-verif` repo).

**Trigger:** module integration-ready (§13) and architect sign-off complete (G4a + G4b).

**Pass criteria** (the verifier accepts the module without architect follow-up):

- Module spec at the design-ready / rtl-ready status, no open architectural questions
- RTL committed at a tagged commit; verifier pins the tag in the regression manifest
- Reference model (for DSP/numerical modules) pinned at the same commit
- Unit TB committed with coverage report green at sign-off thresholds
- Constraints (top-level and module-internal) merged; integration gate evidence archived
- Known issues / waivers documented in the MDS or ADRs

**Evidence:** verifier accepts in the verif repo's regression manifest; design-repo MDS RTM rows updated to `in_verification`.

---

## 15. Hardware bring-up sign-off

Hardware bring-up sign-off has two stages: smoke and functional. Both run on the BittWare RFX-8440A board with the SFU bitstream loaded.

### 15.1 Smoke test

**Pass criteria:**

- Bitstream loads; PS boots; PL identifies via management interface
- All clocks lock; clock telemetry matches FAD §4.1 frequencies within tolerance
- All resets release cleanly; no module reports persistent error state
- Mgmt register read/write round-trip succeeds for at least one register per module
- Event log accessible; reports no critical events at idle
- Temperature and voltage within board operating envelope

### 15.2 Functional bring-up

**Pass criteria:**

- For every row in FAD §12.1a (verification matrix) marked "capture" / "playback/capture" / "spectral check" / "tone check" / "1PPS test" / "stress" / "bring-up": the corresponding hardware test passes
- Recorded loopback / RF-loopback measurements match the architecture-level expected behaviour within the tolerance stated in the relevant MDS §7.3
- 1PPS-aligned scheduled apply: parameter changes commit at the UTC tick boundary, confirmed via event log
- No saturation or error counter increments under nominal traffic
- Backpressure stress: no sample loss under maximum legal stall

### 15.3 Bring-up evidence

All hardware bring-up evidence is captured under `reports/<commit>/hardware/` (gitignored locally; archived by CI). The MDS §12 latency rows are updated with measured values.

---

## 16. Release candidate checklist

A release candidate is the FPGA bitstream plus all evidence required to ship to system integration. The checklist below is run end-to-end at each RC promotion (CI Level 4 — see [04-execution-flow.md](04-execution-flow.md) §15).

- [ ] Documentation gate clean (§3) — every `arch/` document at its target lifecycle state
- [ ] Every module's G4a and G4b passed (§12)
- [ ] Every module's integration gate passed (§13)
- [ ] Top-level synthesis sign-off (§8)
- [ ] CDC sign-off at top level (§7)
- [ ] Constraints sign-off (§10)
- [ ] Implementation sign-off (§9)
- [ ] DV handoff (§14) at the latest tagged commit
- [ ] Hardware bring-up smoke + functional sign-off (§15)
- [ ] RTM: all rows with status `closed` / `in_verification` carry evidence; remaining `open` rows justified
- [ ] Release notes drafted (separate doc; out of scope here)
- [ ] Bitstream identification fields (USR_ACCESS, SLR-aware) verified

A failed checkbox blocks RC promotion. Waiving a checkbox requires an ADR and is not a routine path.

---

## 17. LLM-assisted usage

Claude-assisted usage policy (drafting, reviewing, repo operations) lives in [04-execution-flow.md](04-execution-flow.md) §16. It is not duplicated here. Sign-off remains a human responsibility regardless of which tool drafted, reviewed, or summarised an artefact.

---

## 18. References

- [01-fpga-project-methodology.md](01-fpga-project-methodology.md) — top-level methodology and review-gate overview
- [02-architect-workflow.md](02-architect-workflow.md) — architect-side gate detail (G1–G4a)
- [03-rtl-design-workflow.md](03-rtl-design-workflow.md) — designer-side gate detail (G4b, integration)
- [04-execution-flow.md](04-execution-flow.md) — gate commands, report locations, IP / waiver / archival policy
- [../../arch/README.md](../../arch/README.md) — document lifecycle states, gates G1–G5
- [../../constraints/README.md](../../constraints/README.md) — XDC structure, exception fields
- [../../waivers/README.md](../../waivers/README.md) — waiver discipline
- UG949 — AMD UltraFast Design Methodology
- UG903 — AMD Vivado Using Constraints
- UG906 — AMD Vivado Design Analysis and Closure
- UG908 — AMD Vivado Programming and Debugging

---

## 19. Open items

| ID | Item | Owner | Expiry trigger | Status |
|---|---|---|---|---|
| SIGNOFF-001 | Decide whether MDS frontmatter `implementation_evidence: complete` flag is adopted (Change 6 in revision plan) | Architect | First module reaches §13 integration gate | open |
| SIGNOFF-002 | Pin per-resource utilisation hard-fail thresholds (§8, §9) | Architect | First top-level synthesis | open |
| SIGNOFF-003 | Define hardware bring-up test plan template under `tests/hardware/` (or equivalent) | RTL Designer + Verifier | First hardware delivery | open |
| SIGNOFF-004 | Resolved by [ADR-0005](../../arch/adr/0005-module-spec-uarch-split.md): `design_ready` is canonical; `rtl_ready` retired. ADR currently in `proposed` state pending review. | Architect | ADR-0005 acceptance | resolved |
