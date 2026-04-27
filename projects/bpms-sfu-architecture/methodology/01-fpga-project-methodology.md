---
doc_id: BPMS-SFU-METH-001
doc_type: methodology
project: bpms-sfu-fpga-design
status: draft
version: 0.1
date: 2026-04-26
author: Marco Pausini
---

# FPGA Project Methodology

This document is the starting point and core reference for any engineer working on the
BPMS 1.0 SFU FPGA project. It defines the roles, phases, document types, tool flow, and
conventions that govern the project from architecture through hardware bring-up.

The project produces the PL RTL design for the Satellite Frontend Unit (SFU) on the
BittWare RFX-8440A (AMD Zynq UltraScale+ RFSoC XCZU43DR-2FFVE1156E). PS software,
system-level integration, and UVM/integration verification are out of scope for this
repo — see §6 for companion repositories.

---

## 1. Roles

Three engineering roles operate across the project lifecycle. Each role has a detailed
workflow document.

| Role | Responsibility | Detailed workflow |
|---|---|---|
| **FPGA Architect** | Define device-level architecture, module decomposition, interfaces, clocking, budgets, and management. Author FAD, module specs (`specs.md`), ICDs, ADRs. Author reference models for DSP/numerical modules. Own review gates G1–G3, G4a. | [architect-workflow.md](architect-workflow.md) |
| **FPGA RTL Designer** | Implement RTL from baselined module specs. Author micro-architecture (`uarch.md`), unit testbenches, and non-DSP reference models. Own module-level lint, simulation, and OOC synthesis. Own review gates G4b, integration gate. | [rtl-design-workflow.md](rtl-design-workflow.md) |
| **FPGA Verification Engineer** | Own integration and top-level verification (UVM, subsystem simulation, coverage closure). Consume `arch/` as read-only reference. Own review gate G5. | Lives in `bpms-sfu-fpga-verif` repo |

A single engineer may fill multiple roles. The role boundaries define *what work products are owned*, not org-chart positions.

### 1.1 Human authority

The responsible engineer is the final decision authority for every role. LLM tools
(Claude, ChatGPT, Claude Code) assist with drafting, review, consistency checking, and
repo operations, but do not sign off on any artifact. See §5 for the LLM-assisted
workflow.

---

## 2. Project phases

The project follows a phased flow. Each phase produces artifacts that are consumed by the
next. Phases may overlap — a later phase can start on modules whose inputs are ready while
earlier-phase work continues on other modules.

```
Phase 1 — Architecture
  Inputs:  system architecture docs (ARCH-001, SFU-001)
  Outputs: FAD, core ICDs, ADRs
  Gates:   G1 Orient, G2 Contracts, G3 Budgets

Phase 2 — Module specification
  Inputs:  FAD, ICDs, ADRs
  Outputs: module specs (design_ready), reference models (DSP), RTM
  Gates:   G4a Spec sign-off (per module)

Phase 3 — RTL implementation
  Inputs:  baselined module specs, frozen ICDs, reference models
  Outputs: micro-architecture (uarch.md), RTL source, unit TBs, lint/sim/OOC reports
  Gates:   G4b uArch/RTL sign-off, integration gate (per module)

  RTL implementation is the full module-level engineering cycle: micro-architecture,
  RTL coding, unit testbench authoring, reference model correlation, lint, simulation,
  and out-of-context synthesis. All of these are owned by the RTL designer.

Phase 4 — Integration and implementation
  Inputs:  integration-ready modules, top-level constraints
  Outputs: top-level synthesis, implementation, timing/CDC/power reports
  Gates:   implementation sign-off

Phase 5 — Hardware bring-up
  Inputs:  bitstream, bring-up checklist
  Outputs: smoke test evidence, functional validation evidence
  Gates:   hardware bring-up sign-off
```

Detailed gate definitions: see [signoff-criteria.md](signoff-criteria.md).
Tool commands and report requirements: see [execution-flow.md](execution-flow.md).

---

## 3. Document types

The project uses a 5-document architecture/specification framework plus methodology
documents. Architecture documents live in `arch/`. Methodology documents live in
`docs/methodology/`.

### 3.1 Architecture documents (in `arch/`)

| Type | Acronym | Filename / location | Purpose | Count | Lifetime |
|---|---|---|---|---|---|
| **FPGA Architecture Document** | FAD | `arch/fad/FAD.md` | Device-level architecture: module decomposition, clocking, CDC, memory, interfaces, fixed-point policy, budgets | 1 | Baselined at milestone |
| **Module Spec** | MS | `arch/modules/<module_name>.md` | Per-module black-box contract: ports, interfaces, clock domains, registers, operation, performance, errors, test hooks | 1 per functional block | Baselined before RTL starts |
| **Interface Control Document** | ICD | `arch/icd/<interface_name>.md` | Shared protocols and interfaces used across modules | Few (~5) | Frozen before dependent specs |
| **Architecture Decision Record** | ADR | `arch/adr/NNNN-<kebab-title>.md` | "Why" behind non-trivial choices | As needed | Immutable once accepted |
| **Requirements Traceability Matrix** | RTM | `arch/rtm.md` | Requirement → module → test linkage | 1 (living) | Updated continuously |

**Module Spec vs Micro-Architecture.** The Module Spec (MS) and the Micro-Architecture
(uArch) are two distinct document types with different owners, even though both happen
to be Markdown:

| | Module Spec (MS) | Micro-Architecture (uArch) |
|---|---|---|
| Filename | `arch/modules/<module_name>.md` | designer's choice (e.g., `rtl/<module>/uarch.md`) |
| Owner | Architect | RTL Designer |
| Scope | External contract: what the block does | Internal implementation: how it does it |
| Repo location | `arch/` | alongside RTL |
| Lifecycle gate | G4a (`design_ready`, see [arch/modules/_template.md](../../arch/modules/_template.md) §11.5 self-check) | G4b (uarch authored + module-spec §12.3 implementation evidence complete) |

The MS lives in the architect's `arch/` folder. The uArch lives in the designer's
RTL workflow and is not an architect deliverable. See
[rtl-design-workflow.md](rtl-design-workflow.md).

Templates, conventions, lifecycle states, and review gates are defined in
[arch/README.md](../../arch/README.md) (the framework definition).

### 3.2 Methodology documents (in `docs/methodology/`)

| Document | Purpose |
|---|---|
| **fpga-project-methodology.md** (this file) | Top-level entry point: roles, phases, structure |
| **[architect-workflow.md](architect-workflow.md)** | Detailed FPGA architect workflow including LLM-assisted process |
| **[rtl-design-workflow.md](rtl-design-workflow.md)** | Detailed RTL designer workflow |
| **[execution-flow.md](execution-flow.md)** | Tool flow: Vivado commands, CI gates, report requirements, IP policy |
| **[signoff-criteria.md](signoff-criteria.md)** | Sign-off criteria for every gate level |

### 3.3 Authoring order

Architecture documents are authored in dependency order — each layer must be complete
enough that the next consumer can work without guessing.

1. **FAD §1–§6** — scope, boundary, decisions, block diagram, dataflow, clocking, module inventory
2. **Core ICDs** — streaming bus, register bus, OBG frame protocol
3. **FAD §7–§12** — interface conventions, fixed-point policy, budgets, debug, management, verification contract
4. **Module specs** — one per functional block in the FAD §6 inventory
5. **RTM** — seeded on day one; updated continuously

---

## 4. Repository layout

```
bpms-sfu-fpga-design/
├── arch/                # specifications: FAD, module specs, ICD, ADR, RTM
│   ├── README.md        # framework definition (conventions, lifecycle, gates)
│   ├── fad/
│   ├── modules/         # module specs (<module_name>.md per functional block)
│   ├── icd/
│   ├── adr/
│   └── rtm.md
├── rtl/                 # SystemVerilog sources, one directory per module
├── model/               # Python bit-exact reference models
├── tb/                  # unit testbenches (cocotb / SV), per module
├── constraints/         # XDC files + constraints README
│   ├── README.md        # XDC structure, naming, ownership, exception discipline
│   └── ooc/             # out-of-context module constraints
├── prj/                 # Vivado project, IP (.xci), Tcl recreation scripts
│   └── ip/              # vendor IP configurations, version-controlled
├── scripts/             # Makefile, Tcl, CI, codegen, lint, doc_check
├── docs/                # rendered docs, diagrams
│   └── methodology/     # this folder: workflow and process documents
├── reports/             # tool-generated evidence (gitignored or CI-archived)
└── waivers/             # lint, CDC, timing waivers (version-controlled)
    ├── lint/
    ├── cdc/
    └── timing/
```

Layout principles:
- **Flat by artefact type**, not by lifecycle phase.
- **Design and verification are separate repos.** `bpms-sfu-fpga-verif` consumes `arch/` read-only and pins design-repo commits per regression.
- **Reference models live in the design repo** — they are part of the specification, authored before RTL.
- **Unit testbenches live in the design repo** — authored alongside module specs and RTL. Integration/UVM testbenches live in the verif repo.
- **Reports are not committed to main** — generated by CI or local builds, archived by commit hash.
- **Waivers are version-controlled** — they are design decisions, not transient notes.

---

## 5. LLM-assisted workflow

The project uses LLM tools to accelerate architecture, specification, and implementation
work. Each role has its own LLM-assisted workflow.

### 5.1 Architect LLM workflow

Three LLM roles are defined for the architecture phase:

| LLM role | Tool | Purpose | Does NOT do |
|---|---|---|---|
| `fpga_arch` (drafter) | Claude.ai | Draft FAD/spec/ICD/ADR sections from source docs and architect notes | Invent requirements; resolve TBDs without source; sign off |
| `fpga_arch_reviewer` (reviewer) | ChatGPT or independent LLM | Adversarial review: find gaps, check methodology, challenge assumptions | Rewrite documents; make architecture decisions |
| `repo_operator` (executor) | Claude Code | Repo-aware edits: apply accepted changes, run consistency checks, generate scripts | Change frozen/baselined docs without instruction; silently resolve markers |

The pipeline:

```
Draft (Claude) → Review (ChatGPT) → Decide (Engineer) → Apply (Claude Code) → Check
```

Every step has a clear handoff. The engineer decides which review items are accepted,
rejected, or deferred. LLM-generated sign-off summaries are advisory only.

Detailed architect LLM workflow, including reviewer prompt structure, review cadence,
acceptance protocol, and conflict resolution: see
[architect-workflow.md](architect-workflow.md) §6.

### 5.2 RTL designer LLM workflow

[STUB: To be authored by the RTL designer in [rtl-design-workflow.md](rtl-design-workflow.md).
The RTL designer uses Claude Code (or equivalent) for LLM-assisted RTL generation from
the Module Spec. This subsection will summarize the role split and tool usage at the
designer level, with details in the rtl-design-workflow document.]

### 5.3 Anti-laundering discipline

The primary risk in multi-LLM workflows is polished but wrong output — one LLM invents,
another validates based on the same incomplete context. This applies across all roles.
Countermeasures:

- Every factual claim traces to a source document with section citation
- Every inference carries `[INFERRED from <source §>]`
- Every assumption carries `[ASSUMPTION: <text>, <expiry trigger>]`
- Every architecture decision has an ADR
- Every "ready" state requires execution evidence, not just document completeness

---

## 6. Companion repositories

| Repository | Scope | Relationship |
|---|---|---|
| `bpms-sfu-fpga-design` (this repo) | PL RTL design: architecture, specs, RTL, unit TBs, reference models, constraints | Primary |
| `bpms-sfu-fpga-verif` (planned) | Integration/UVM verification, coverage, regressions | Consumes `arch/` read-only; pins design-repo commit |
| PS software repo (TBD) | TLE/SGP4, mgmt protocol stack, Ethernet GEM driver, NMS logic | Interfaces defined by PL-side `mgmt` and `band_doppler` module specs |

---

## 7. Key conventions

These conventions apply across all documents and roles. Full details in
[arch/README.md](../../arch/README.md).

### 7.1 Placeholder markers

| Marker | Meaning |
|---|---|
| `[TBD: <reason>, <owner>]` | Value not yet known; names who owns the resolution |
| `[STUB: <blocking item>]` | Section deliberately empty; names the blocker |
| `[ASSUMPTION: <text>, <expiry trigger>]` | Chosen without source confirmation; names what will confirm or overturn |
| `[INFERRED from <source §>]` | Derived from a source doc but not literally stated there |

### 7.2 Citation discipline

Every factual claim traceable to a parent document carries an inline citation:
`(ARCH-001 §5.2)` or `(SFU-001 §6.4)`. Claims without citation are treated as derived
or proposed and must carry the matching marker.

### 7.3 Source of truth

- **Primary:** BPMS 1.0 SFU Architecture (BPMS-1.0-SFU-001)
- **Secondary:** BPMS 1.0 System Architecture (BPMS-1.0-ARCH-001)
- When documents conflict, prefer the more specific and more recent.
- Do not invent missing requirements.

### 7.4 Design-ready criteria

A module spec (`spec.md`) is `design_ready` when the external contract is complete and
unambiguous — sufficient for a senior RTL designer to start implementation:

- Clock domains: list of input clocks and resets, with rate/range and reset style
- Interfaces: type (AXI4-Full/Lite/Stream, custom), clock domain, and for non-standard interfaces a waveform or timing description
- Parameters: name, default, valid range, effect
- Register map: minimum set of registers (high-level; address/bit assignment is designer's scope)
- Operation: configuration sequence, steady-state behavior, mode transitions, shutdown
- Performance targets: throughput, latency (as budgets, not implementation-prescriptive)
- Error and status behavior: what is reported, how
- System-level constraints: CDC mechanism requirements, resource ceiling, fixed-point format at boundaries
- Test plan hooks: observable behaviors that DV must cover, mapped to requirements
- Reference model pointer (for DSP/numerical modules): acceptance criterion

A module spec does NOT prescribe internals: FSMs, pipeline depth, FIFO implementation,
internal sub-block decomposition, or internal fixed-point precision. Those are the
designer's micro-architecture decisions.

The test: could a senior RTL designer implement this module from the spec + referenced
ICDs, making their own micro-architecture choices, without asking the architect a question
about intended *external* behavior?

### 7.5 Review gates

| Gate | Scope | Owner |
|---|---|---|
| **G1 Orient** | FAD §1–§6: scope, boundary, block diagram, clocking, module inventory | Architect |
| **G2 Contracts** | FAD §7–§8, core ICDs: interface conventions, fixed-point policy, ICDs frozen | Architect |
| **G3 Budgets** | FAD §9: latency, resource, floorplan, power — all populated or TBD with owner | Architect |
| **G4a Spec sign-off** | Per module spec: `design_ready` criteria met, reference model provided (if numerical), test hooks defined | Architect |
| **G4b uArch/RTL sign-off** | Per module: uarch complete, RTL passes lint/sim/OOC, refmodel correlated | RTL designer |
| **G5 Handoff to DV** | Baselined spec + RTL + TB: DV can consume without architect follow-up | Verification engineer |

No gate may be waived silently. Waivers are recorded as ADRs.

---

## 8. AMD/Xilinx methodology alignment

This project methodology is built on AMD's UltraFast Design Methodology (UG949) and
companion guides. The execution flow and sign-off criteria operationalize these references
into project-specific gates.

| AMD reference | Project coverage |
|---|---|
| UG949 — UltraFast Design Methodology | [execution-flow.md](execution-flow.md), [signoff-criteria.md](signoff-criteria.md) |
| UG903 — Using Constraints | [constraints/README.md](../../constraints/README.md), signoff §6 |
| UG906 — Design Analysis and Closure | execution-flow §8–§12, signoff §8–§9 |
| UG900 — Logic Simulation | execution-flow §7, signoff §4–§5 |
| UG908 — Programming and Debugging | signoff §11 (hardware bring-up) |

---

## 9. How to get started

**If you are the FPGA architect:** Read [architect-workflow.md](architect-workflow.md).
Start with FAD §1–§6, then core ICDs, then FAD §7–§12, then module specs.

**If you are an RTL designer:** Read [rtl-design-workflow.md](rtl-design-workflow.md).
Your inputs are a baselined module spec (`spec.md`), frozen ICDs, and the FAD. Your
outputs are micro-architecture (`uarch.md`), RTL, unit TBs, and implementation evidence.

**If you are a verification engineer:** Read the verification workflow in the
`bpms-sfu-fpga-verif` repo. Your inputs are baselined module specs, RTL, unit TBs, and
reference models from this repo.

**If you need to understand the tool flow:** Read
[execution-flow.md](execution-flow.md) for Vivado commands, CI gates, and report
requirements.

**If you need to know what "done" means:** Read
[signoff-criteria.md](signoff-criteria.md) for sign-off criteria at every level.

---

## 10. References

| Ref | Document | ID |
|---|---|---|
| [P1] | BPMS 1.0 System Architecture | BPMS-1.0-ARCH-001 v2.4 |
| [P2] | BPMS 1.0 SFU Architecture | BPMS-1.0-SFU-001 v1.6 |
| [P3] | FPGA Architecture Framework | fpga-arch-framework.md |
| [P4] | FPGA Project Best-Practice Methodology | fpga-prj-best-methodology.md |
| [UG949] | AMD UltraFast Design Methodology Guide | docs.amd.com |
| [UG903] | AMD Vivado Using Constraints | docs.amd.com |
| [UG906] | AMD Vivado Design Analysis and Closure | docs.amd.com |
| [UG900] | AMD Vivado Logic Simulation | docs.amd.com |
| [UG908] | AMD Vivado Programming and Debugging | docs.amd.com |