---
doc_id: BPMS-SFU-METH-002
doc_type: methodology
project: bpms-sfu-fpga-design
status: draft
version: 0.1
date: 2026-04-26
author: Shlomi Kulik
---

# FPGA Architect Workflow

This document defines the detailed workflow for the FPGA architect role on the BPMS 1.0
SFU FPGA project. It covers the architect's responsibilities, the document authoring
process, the LLM-assisted drafting and review pipeline, review gates, and handoff to RTL
design.

Parent document: [fpga-project-methodology.md](fpga-project-methodology.md).

---

## 1. Role definition

The FPGA architect owns the device-level architecture of the SFU FPGA. This means
defining *what* blocks exist, *how* they connect, and *why* — before anyone writes RTL.

### 1.1 Responsibilities

- Define the SFU FPGA module decomposition, signal chain, and top-level block diagram
- Define clocking, reset, and CDC architecture
- Define internal interface conventions and freeze core ICDs
- Define fixed-point policy and numerical precision at every stage boundary
- Allocate latency, resource, floorplan, and power budgets
- Define the debug, observability, and management architecture
- Author the FAD, core ICDs, key ADRs, and module specs (`spec.md`) for each functional block
- Author reference models (Python bit-exact) for DSP and numerically critical modules
- Define unit-TB verification scenarios, coverage goals, and test hooks in each module spec
- Maintain architecture invariants and the functional boundary (DCU/SFU ownership)
- Derive FPGA-level requirements from system documents; maintain the RTM
- Review module specs for architectural consistency before handoff to designer
- Own review gates G1 (Orient), G2 (Contracts), G3 (Budgets), and G4a (Spec sign-off)

### 1.2 Does not own

- Module micro-architecture: FSMs, pipeline depth, FIFO implementation, internal sub-block decomposition (RTL designer)
- Internal fixed-point precision (only boundary formats are architect-specified)
- Register bit-level layout (architect specifies minimum register set; designer assigns addresses/bits)
- Per-module RTL implementation (RTL designer)
- Unit testbench implementation (RTL designer; architect defines scenarios and hooks)
- Integration/UVM verification (verification engineer, verif repo)
- Vivado implementation closure (shared with RTL designer)
- PS software (separate repo)

### 1.3 Inputs

- BPMS 1.0 System Architecture (ARCH-001 v2.4)
- BPMS 1.0 SFU Architecture (SFU-001 v1.6)
- Filter Bank Design Specification (AST ref [7])
- Target device datasheet (AMD XCZU43DR)
- BittWare RFX-8440A product datasheet
- Architect's own domain knowledge and engineering judgment

### 1.4 Outputs

- FAD — FPGA Architecture Document (1)
- Module specs (`spec.md`) — one per functional block in the FAD §6 inventory
- ICDs — Interface Control Documents (~5)
- ADRs — Architecture Decision Records (as needed)
- RTM — Requirements Traceability Matrix (1, living)
- Reference models (Python) — for DSP and numerically critical modules
- Architecture invariants, budgets, and derived requirements

---

## 2. Architecture phase workflow

The architect works through the FAD in dependency order. Each step produces a section or
group of sections that are reviewed, revised, and committed before the next step begins.

### 2.1 Phase 1 — Orient (FAD §1–§6)

**Goal:** Establish the FPGA's scope, functional boundary, key decisions, top-level
structure, dataflow, clocking architecture, and module inventory.

| Step | FAD section | Key outputs |
|---|---|---|
| 1 | §1 Scope and Context | Functional boundary table, invariants, key decisions, parent doc refs |
| 2 | §2 Top-Level Block Diagram | DL datapath, UL datapath, infrastructure views |
| 3 | §3 Dataflow | Stage-by-stage narrative for DL, UL, and debug/auxiliary paths |
| 4 | §4 Clocking and Reset Architecture | Clock inventory, clock topology, primary/secondary modes, reset topology, CDC inventory |
| 5 | §5 Memory Architecture | On-chip memory allocation (BRAM, URAM), external memory (if any) |
| 6 | §6 Module Inventory | Canonical list of every functional block with module spec link, reuse type, clock domains, owner |

**Review gate:** G1 Orient — all §1–§6 sections consistent; module inventory drives every
module spec filename that follows.

**ADR triggers:** Clock architecture (ADR-0002), any non-obvious module boundary or
ownership decision.

### 2.2 Phase 1 (continued) — Contracts (FAD §7–§8, core ICDs)

**Goal:** Define the interface conventions and fixed-point policy that every module must
follow. Freeze core ICDs so module spec authors have stable contracts.

| Step | Output | Key outputs |
|---|---|---|
| 7 | FAD §7 Internal Interface Conventions | Streaming data convention, register/control convention, backpressure rules |
| 8 | FAD §8 Fixed-Point and Numerical Policy | Q-format per stage, growth/rounding/saturation rules, reference model policy |
| 9 | ICD: streaming_bus | Signal list, payload format, sideband format, timing, assertions |
| 10 | ICD: register_bus | Protocol, address map, access width, CDC rules |
| 11 | ICD: obg_frame | OBG Aurora frame structure, bin layout, lane protocol |

**Review gate:** G2 Contracts — interface conventions complete; core ICDs frozen.

**ADR triggers:** Bus protocol choice (ADR if non-obvious), PS-PL split (ADR-0003).

### 2.3 Phase 1 (continued) — Budgets and remaining sections (FAD §9–§12)

**Goal:** Allocate budgets, define debug/management architecture, and establish the
verification contract.

| Step | FAD section | Key outputs |
|---|---|---|
| 12 | §9 Budgets | Latency budget, resource budget, floorplan intent, power budget |
| 13 | §10 Debug and Observability Architecture | Minimum telemetry set, capture/playback infrastructure, event log, loopback |
| 14 | §11 Management Plane | Register bus topology, scheduled apply mechanism, autonomous update mechanism, NV config |
| 15 | §12 Architecture-Level Verification Contract | Two-tier model (unit TB + UVM), reference model policy, verification matrix, handoff spec |

**Review gate:** G3 Budgets — all budget cells populated or explicitly TBD with owner.

### 2.4 Phase 2 — Module specification

**Goal:** Produce `design_ready` module specs (`spec.md`) for every functional block in
the FAD §6 inventory.

**Granularity:** The architect specifies at the functional-block level — each spec defines
a black-box contract for a block that may contain multiple .sv files internally. The
designer decides the internal decomposition as part of the micro-architecture. The FAD §6
inventory lists functional blocks, not individual .sv files. The exact partitioning into
blocks is an output of the FAD §2 (block diagram) and §6 (module inventory) authoring
process, reviewed at G1.

For each module spec:

1. Draft from FAD + ICDs + source docs using the module spec template
2. Define external contract: ports, interfaces, clock domains, registers, operation, performance, errors
3. Add system-level constraints: latency budget, CDC mechanism, resource ceiling, fixed-point at boundaries
4. Add reference model pointer (for DSP/numerical modules)
5. Add test plan hooks mapped to requirements
6. Review against `design_ready` criteria (§5 below)
7. Ensure all upstream requirements traced in RTM
8. Commit as `design_ready` only when external contract is complete

**Review gate:** G4a Spec sign-off — per module, `design_ready` criteria met.

### 2.5 Ongoing maintenance

- **RTM:** Update whenever a module is assigned, a unit TB case is added, or UVM coverage
  is confirmed. Claude Code can regenerate and diff mechanically.
- **ADRs:** Author whenever a non-trivial choice is made. Immutable once accepted.
- **FAD updates:** Re-baseline on major change. Track in FAD §15 Change Log.

---

## 3. Section-by-section drafting process

The architect drafts each FAD/module-spec/ICD section individually, not entire documents at once.
This keeps each iteration focused and reviewable.

### 3.1 Drafting checklist (per section)

Before drafting:

1. Identify the target template section (from `arch/` templates)
2. Gather the relevant source material: parent doc sections, existing FAD sections, related ADRs
3. List what is *specified* vs *must be inferred* vs *must be proposed*

During drafting:

4. Fill the template section, citing sources inline
5. Mark every gap: `[TBD]`, `[STUB]`, `[ASSUMPTION]`, `[INFERRED]`
6. Identify ADR triggers (non-trivial choices that need recorded rationale)
7. Identify cross-references to other sections or documents

After drafting:

8. Self-check against the relevant review gate criteria
9. Submit for independent review (§6)
10. Revise based on accepted review items
11. Commit to repo

### 3.2 What "good enough to commit" means

A section is committable when:

- It follows the template structure
- All factual claims have source citations
- All gaps are explicitly marked with the correct placeholder
- It is internally consistent with previously committed sections
- It has been through at least one independent review cycle
- The architect has accepted or rejected all review items

A section does NOT need to be complete to be committed. `[TBD]` and `[STUB]` markers are
expected in early drafts. What matters is that gaps are *visible and owned*, not hidden.

---

## 4. Conventions

### 4.1 Placeholder markers

| Marker | When to use |
|---|---|
| `[TBD: <reason>, <owner>]` | Value not yet known. Name who owns the resolution. |
| `[STUB: <blocking item>]` | Section deliberately empty. Name what blocks it. |
| `[ASSUMPTION: <text>, <expiry trigger>]` | Value chosen without source confirmation. Name what will confirm or overturn. |
| `[INFERRED from <source §>]` | Derived from a source doc but not literally stated there. |

These are greppable. They enforce the specified / inferred / proposed split. A document
with unmarked gaps is worse than a document with many markers — hidden gaps cause
downstream failures.

### 4.2 Citation discipline

Every factual claim traceable to a parent document carries an inline citation:
`(SFU-001 §6.4)` or `(ARCH-001 §10.1)`. Claims without citation must carry the matching
marker (`[INFERRED]` or `[ASSUMPTION]`).

### 4.3 Architecture invariants

The FAD §1.5 defines architecture invariants — constraints that apply across the entire
design. The architect is responsible for maintaining these and ensuring no module spec, ICD, or
ADR violates them. Any proposal that would violate an invariant requires an ADR with
explicit justification.

### 4.4 Functional boundary discipline

The SFU/DCU ownership boundary is load-bearing. Any proposal that crosses the boundary
(e.g., moving a beam-level function into the SFU, or an SFU function into the DCU) must
be flagged immediately and handled via the FAD §1.2 boundary-crossing procedure:

1. Open an ADR
2. Cite the system-level source that would justify the change
3. If no such source exists, the proposal is rejected by default

### 4.5 Derived requirements

When the FPGA architecture introduces implementation requirements not literally stated in
the parent system documents, the architect creates derived requirements:

- ID format: `FAD-DER-<CATEGORY>-NNN`
- Each must cite the parent requirement, ICD, ADR, or FAD section that forced it
- Each must appear in the RTM
- Each must map to at least one verification target
- Derived requirements cannot silently alter SFU functional ownership

---

## 5. Design-ready criteria and module spec quality

### 5.1 Design-ready checklist

A module spec (`spec.md`) is `design_ready` when the external contract is complete and
unambiguous. Each unchecked item must be listed in frontmatter `design_ready_blocking:`.

- [ ] Clock domains listed with rate/range, reset name, reset style
- [ ] All interfaces listed with type (AXI4-Full/Lite/Stream, custom) and clock domain
- [ ] Non-standard interfaces have waveform or timing description
- [ ] Parameters have name, default, valid range, and effect
- [ ] Minimum register set defined (high-level; address/bit layout is designer's scope)
- [ ] Operation described: config sequence, steady-state, modes, shutdown
- [ ] Performance targets stated: throughput, latency budget
- [ ] Error and status behavior defined
- [ ] System-level constraints stated: CDC mechanism, resource ceiling, fixed-point at boundaries
- [ ] Test plan hooks defined and mapped to requirements
- [ ] Reference model provided (for DSP/numerical modules)
- [ ] Traceability to SYS-* / SFU-* / FAD-DER-* requirements complete

### 5.2 What the spec does NOT prescribe

A module spec defines the external contract. It does NOT prescribe:

- FSMs or state machine structure
- Pipeline depth or staging
- FIFO implementation details (depth may be constrained as a performance target)
- Internal sub-block decomposition (how many .sv files)
- Internal fixed-point precision (only boundary formats)
- Register address/bit assignment (architect specifies minimum set; designer assigns layout)

These are the designer's micro-architecture decisions, documented in `uarch.md`.

### 5.3 The completeness test

Could a senior RTL designer implement this module from the spec + referenced ICDs, making
their own micro-architecture choices, without asking the architect a question about
intended *external* behavior?

If the answer is no, the spec is `not_design_ready`.

### 5.4 Common spec failure modes

| Failure | Symptom | Fix |
|---|---|---|
| Ambiguous interface behavior | "Handles backpressure" without specifying what happens | Add timing diagram or prose: stall, drop, or error? |
| Missing operating mode description | Mode listed but no config sequence or transition rules | Add operation section prose |
| Missing performance target | "Low latency" without a number | State budget in cycles or ns |
| Unspecified error response | Error conditions listed but external behavior undefined | Specify: what status/interrupt, what happens to output |
| No test plan hooks | No observable behaviors mapped to requirements | Add test plan hooks section |
| Internal micro-architecture leaking in | Spec prescribes FSM states or pipeline depth | Remove — state the external constraint, not the implementation |

---

## 6. LLM-assisted workflow

### 6.1 Roles

| Role | Tool | Purpose |
|---|---|---|
| `fpga_arch` (drafter) | Claude.ai | Draft architecture document sections from source docs and architect notes |
| `fpga_arch_reviewer` (reviewer) | ChatGPT or independent LLM | Adversarial review: find gaps, check methodology, challenge assumptions |
| `repo_operator` (executor) | Claude Code | Repo-aware operations: apply accepted changes, run consistency checks, generate scripts |

Role definitions are maintained in the respective tool configurations (Claude Project
instructions, ChatGPT custom instructions, Claude Code CLAUDE.md). They are not committed
to the design repo — they are tool configuration, not design artifacts.

### 6.2 The pipeline

```
Step A — Draft (Claude.ai)
  Input:  relevant parent doc sections, architect notes, target template section
  Output: one FAD/spec/ICD section with markers and citations
  Rule:   do not invent requirements; mark all inferences and assumptions

Step B — Review (ChatGPT)
  Input:  drafted section + framework reference + FAD context + target gate criteria
  Output: issue table (severity / location / issue / action / gate)
  Rule:   do not rewrite; produce issues, not prose

Step C — Decide (Engineer)
  Input:  review issue table
  Output: each item marked accepted / rejected / deferred
  Rule:   rejected items require one-line rationale; factual disputes trace to sources

Step D — Apply (Claude Code or manual edit)
  Input:  accepted review items + current repo state
  Output: updated Markdown committed to repo
  Rule:   minimal diffs; do not change frozen/baselined docs without instruction

Step E — Check (Claude Code)
  Input:  updated repo
  Output: consistency report (cross-references, frontmatter, markers, RTM)
  Rule:   report findings, do not auto-fix
```

### 6.3 Drafter guidelines (Claude.ai / `fpga_arch`)

When drafting a section:

- Work on one section at a time, not entire documents
- Always have the relevant source material in context (parent doc sections, existing FAD, ICDs)
- Produce Markdown ready to paste into the repo template
- Separate specified / inferred / proposed with markers
- Cite sources inline for every factual claim
- End each draft with an unresolved-items table listing TBDs, assumptions, and ADR triggers
- Do not fill TBDs with plausible-sounding values — leave them marked
- For module specs: describe external behavior only; do not prescribe internal micro-architecture

### 6.4 Reviewer guidelines (ChatGPT / `fpga_arch_reviewer`)

The reviewer operates with this context loaded:

1. `fpga-arch-framework.md` (framework conventions, templates, lifecycle)
2. `signoff-criteria.md` (sign-off criteria for the target gate)
3. Current FAD (for cross-consistency)
4. The section under review

The reviewer checks against:

1. Framework conventions (markers, citations, frontmatter)
2. Sign-off criteria for the target gate
3. AMD methodology (UG949, UG903, UG906)
4. Internal consistency with FAD and existing ICDs
5. Design-readiness criteria (if reviewing a module spec)
6. Missing CDC / constraints / timing / verification / debug implications
7. Assumptions incorrectly presented as facts
8. Unverifiable or uncited claims

Reviewer output format:

| Severity | Location | Issue | Action | Gate |
|---|---|---|---|---|
| blocker | FAD §4.5 | CDC from Aurora to DSP clock: mechanism unspecified | Add async FIFO row with depth, reset, owner module spec | G1 / CDC |
| high | spec §Performance | Band gain output Q-format undefined | Specify Q(a.b) and saturation rule | G4a |
| medium | spec §Operation | Reset-while-busy behavior missing | Add drop/complete/error specification | G4a |
| low | FAD §9.1 | Latency budget row for `cw_beacon` missing | Add estimate or TBD with owner | G3 |

The reviewer does NOT rewrite sections unless explicitly asked. The default output is
the issue table.

### 6.5 Review cadence

Reviews happen at defined points, not on every draft iteration:

| Document type | Review trigger | Reviewer focus |
|---|---|---|
| FAD §1–§6 | After §-group completion | G1 criteria: scope, boundary, block diagram, clocking, module inventory |
| Core ICDs | Before freezing | Protocol completeness, assertion coverage, consumer list |
| FAD §7–§12 | After §-group completion | G2/G3 criteria: conventions, budgets, verification contract |
| Module spec | Before `design_ready` transition | Full design-ready checklist, cross-consistency with FAD and ICDs |
| ADR | Before `accepted` transition | Alternatives considered, rationale grounded in forces, consequences honest |
| RTM | Spot-check at each gate; full review at release candidate | All rows mapped, evidence present for closed items |

Do not review every draft iteration — that creates noise. Review the output that will be
committed.

### 6.6 Review acceptance protocol

1. Reviewer produces issue table (§6.4 format)
2. Architect marks each row: `accepted` / `rejected` / `deferred`
3. **Rejected** items require a one-line rationale
4. **Accepted** items become direct edits, new TBD markers with owner, or ADR triggers
5. **Deferred** items move to FAD §13 or module spec Open Questions with expiry trigger
6. The resolved review table is archived as a PR comment or in review notes

When drafter and reviewer disagree on a *factual* point (not style), the resolution must
trace to a source document, a measurement, or an ADR. "I prefer this version" is not a
valid resolution for a factual conflict.

### 6.7 Repo operator tasks (Claude Code)

Valuable repo-aware tasks for the architecture phase:

| Task | Example command |
|---|---|
| Cross-reference check | "List all module specs that reference ICD streaming_bus but streaming_bus.md doesn't list them as consumers" |
| Marker audit | "Find all TBD markers without owner in arch/" |
| RTM regeneration | "Regenerate RTM from FAD + module specs and diff against current rtm.md" |
| Module inventory sync | "Check that every module in FAD §6 has a corresponding file in arch/modules/" |
| CDC inventory sync | "Check that every CDC mentioned in module specs appears in FAD §4.5" |
| Template scaffolding | "Create module spec skeleton for `band_gain` from the template, pre-filling identity from FAD §6" |
| Frontmatter validation | "Find any module spec marked design_ready with non-empty design_ready_blocking" |

Claude Code operates on the actual repo and reports findings. It does not silently resolve
markers or make architecture decisions.

---

## 7. Review gates — architect responsibilities

### 7.1 G1 Orient

**Trigger:** FAD §1–§6 complete.

**Architect checks:**

- Functional boundary table complete; ownership clear for every function
- Architecture invariants defined
- Top-level block diagram matches module inventory (every block has a row)
- Dataflow narrative covers DL, UL, and debug/auxiliary paths
- Clock inventory covers all domains; CDC inventory covers all crossings
- Module inventory is canonical — every functional block has exactly one row

**Evidence:** Reviewed FAD §1–§6 committed to repo. Review issue table resolved.

### 7.2 G2 Contracts

**Trigger:** FAD §7–§8 complete; core ICDs ready to freeze.

**Architect checks:**

- Streaming interface convention is unambiguous (signal list, backpressure, framing)
- Register/control convention is unambiguous (protocol, address width, CDC rules)
- Fixed-point policy covers every stage boundary in the dataflow
- Core ICDs (streaming_bus, register_bus, obg_frame) are complete with protocol assertions
- All consumers listed in each ICD

**Evidence:** Reviewed FAD §7–§8 and ICDs committed. ICDs transition to `frozen`.

### 7.3 G3 Budgets

**Trigger:** FAD §9–§12 complete.

**Architect checks:**

- Latency budget: every pipeline stage has an estimate or TBD with owner
- Resource budget: every module has an estimate or TBD with owner
- Floorplan intent: SLR topology, RF-facing and transceiver-facing placement documented
- Power budget: preliminary XPE estimate or TBD with owner
- Debug architecture: minimum telemetry set defined; capture/playback points identified
- Management plane: register bus topology, scheduled apply, autonomous update documented
- Verification contract: verification matrix populated; two-tier model defined

**Evidence:** Reviewed FAD §9–§12 committed. All budget cells populated or explicitly TBD
with owner and expiry trigger.

### 7.4 G4a Spec sign-off (architect's gate)

The architect authors the module spec. Before `design_ready` transition:

**Architect self-checks:**

- External contract is complete: ports, interfaces, clocks, registers, operation, performance, errors
- Spec is consistent with FAD architecture (module boundary, interfaces, clock domains)
- Spec is consistent with frozen ICDs
- Fixed-point format at boundaries matches FAD §8 policy
- CDC constraints are consistent with FAD §4.5
- All architecture invariants respected
- System-level constraints (latency budget, resource ceiling) are stated
- Derived requirements properly traced in RTM
- Reference model provided for DSP/numerical modules
- Test plan hooks map to requirements
- No internal micro-architecture prescribed (FSMs, pipeline, FIFO implementation)

**Evidence:** Reviewed module spec committed to repo with `status: design_ready`.

### 7.5 Architect's review role at G4b

The designer owns G4b (uarch/RTL sign-off). The architect reviews only for:

- Architectural consistency: does the implementation respect the spec contract?
- CDC: does the implementation match the CDC mechanisms specified in the spec and FAD?
- No undocumented functional boundary crossing
- Resource usage within the budgeted ceiling

---

## 8. Handoff to RTL design

When a module spec reaches `design_ready`:

1. Module spec is committed with `status: design_ready` and empty `design_ready_blocking`
2. Referenced ICDs are `frozen`
3. Referenced ADRs are `accepted`
4. Reference model exists (for DSP/numerical modules)
5. RTM row maps to the module
6. System-level constraints are stated (latency budget, CDC mechanism, resource ceiling, fixed-point at boundaries)

The RTL designer's input set:

```
Module spec (spec.md, baselined, design_ready)
+ referenced ICDs (frozen)
+ FAD (for system context: clocking, conventions, budgets)
+ reference model (if DSP/numerical)
```

The designer then produces:

```
Micro-architecture (uarch.md) — internal implementation decisions
+ RTL source (.sv files)
+ unit testbenches
+ implementation evidence (lint, sim, OOC reports)
```

The spec defines *what* the module does externally. The uarch defines *how* it does it
internally. If the designer needs to ask the architect a question about intended external
behavior, the spec is not `design_ready` — the answer belongs in the spec.

---

## 9. Architect's role in later phases

### 9.1 During RTL implementation (Phase 3)

- Review architectural consistency of RTL (not code style or synthesis optimization)
- Resolve cross-module integration questions
- Update FAD budgets with post-synthesis actuals
- Author new ADRs for implementation-driven architecture changes

### 9.2 During integration (Phase 4)

- Review top-level constraints against FAD §4 (clocking) and §9.3 (floorplan intent)
- Review CDC reports against FAD §4.5 CDC inventory
- Adjudicate timing closure issues that require architectural change (pipeline insertion, clock domain restructuring)

### 9.3 During hardware bring-up (Phase 5)

- Define bring-up sequence based on FAD §10 debug architecture
- Review hardware validation evidence against verification contract

---

## 10. Open items

| ID | Item | Owner | Expiry trigger | Status |
|---|---|---|---|---|
| ARCH-WF-001 | Define ChatGPT reviewer prompt template with project-specific context loading | Architect | Before first G1 review | open |
| ARCH-WF-002 | Define Claude Code CLAUDE.md with repo-operator role and consistency-check commands | Architect | Before first repo-operator task | open |
| ARCH-WF-003 | Define review archive format (PR comment vs dedicated folder) | Architect | Before first committed review | open |