# FPGA Architecture Framework — Revision Plan v2

**Based on:** `fpga-arch-framework-revision-plan.md` (original plan, ChatGPT-authored)
**Reviewed and revised by:** Claude (Step 3 review, 2026-04-26)
**Purpose:** Concrete change list to extend `fpga-arch-framework.md` from a documentation/specification framework into a complete FPGA project methodology.

---

## Guiding principle

Two layers, not one:

```text
Layer 1 — patch the existing arch/ framework (documents, templates, conventions)
Layer 2 — add a lightweight execution layer (tool flow, gates, sign-off, evidence)
```

The architecture folder (`arch/`) remains the specification source of truth. Execution/methodology documents live in `docs/methodology/`. No new architecture document types — keep the 5-doc framework (FAD / MDS / ICD / ADR / RTM).

---

## Repo layout update

The current layout (`state.md`) does not include several folders implied by the execution layer. Updated layout:

```text
bpms-sfu-fpga-design/
├── arch/              # specifications: FAD, MDS, ICD, ADR, RTM
├── rtl/               # SystemVerilog sources, one directory per module
├── model/             # Python bit-exact reference models
├── constraints/       # XDC files + constraints README
├── prj/               # Vivado project, IP (.xci), Tcl recreation scripts
├── scripts/           # build, CI, codegen, lint, Makefile
├── tb/                # unit testbenches (cocotb/SV), per-module
├── docs/              # rendered docs, diagrams
│   └── methodology/   # flow.md, signoff.md
├── reports/           # tool-generated evidence (gitignored or CI-archived)
└── waivers/           # lint, CDC, timing waivers (version-controlled)
```

**Decisions embedded in this layout:**

- `tb/` (unit testbenches) lives in the design repo — authored alongside MDS and RTL. Integration/UVM testbenches live in the companion `bpms-sfu-fpga-verif` repo.
- `reports/` is gitignored or CI-archived — not committed to main, but reproducible from committed sources.
- `waivers/` is version-controlled — waivers are design artifacts, not transient notes.
- `docs/methodology/` holds process documents, not architecture specs.

---

## Change list

### Change 1 — Rename FAD §12

**Current:** `## 12. Verification Strategy (thin)`

**New:** `## 12. Architecture-Level Verification Contract`

Rationale: "thin" signals optional or weak. The section can be concise, but it defines what the architecture expects unit TBs, DV, and hardware bring-up to prove. That contract is not optional.

**Add to FAD §12:** Architecture-level verification matrix.

| Feature | Unit TB | Subsystem | UVM/top | Hardware |
|---|---|---|---|---|
| OBG frame alignment | yes | yes | yes | capture |
| Bins selector mapping | yes | yes | yes | playback/capture |
| Band gain | bit-true | yes | yes | spectral check |
| Band doppler | bit-true | yes | yes | tone check |
| UTC scheduled apply | yes | yes | yes | 1PPS test |
| Backpressure | yes | yes | yes | stress |
| Reset recovery | yes | yes | yes | bring-up |

This defines *what must be verified*, not *how* (that's DV's job). Populated per-project; the template carries the table structure and a few example rows.

### Change 2 — Add architecture invariants as FAD §1.5

**Location:** After FAD §1.4 (Key Architectural Decisions), before §1.5 (Parent Documents, renumbered to §1.6).

Rationale: invariants constrain the entire design and must be visible early, before the reader dives into dataflow or clocking. Placing them after §3 (as original plan suggested) buries them.

```markdown
## 1.5 Architecture Invariants

| ID | Invariant | Rationale | Checked by |
|---|---|---|---|
| INV-001 | SFU performs band-level processing only; beam-level gain, Doppler, and delay remain outside the SFU unless a boundary-change ADR is accepted. | Preserves functional ownership (SFU-001 §4.2, §4.4). | FAD review, RTM |
| INV-002 | All inter-module streaming interfaces obey the frozen streaming ICD. | Prevents protocol drift. | ICD review, lint/SVA, unit TB |
| INV-003 | No unregistered combinational ready/valid path crosses a module boundary. | Timing closure and composability. | RTL review, lint/SVA |
| INV-004 | Every CDC is listed in FAD §4.5 and implemented using an approved CDC mechanism. | Hardware safety. | CDC review, report_cdc |
| INV-005 | Scheduled configuration updates commit atomically at the defined trigger boundary. | Prevents mixed-parameter operation. | Unit TB, UVM, hardware test |
| INV-006 | DSP modules with non-trivial numerical behavior have a bit-true or explicitly bounded reference model. | Numerical correctness. | Unit TB, refmodel correlation |
```

### Change 3 — Add derived requirements policy to RTM

**Location:** `arch/rtm.md`, new section before the requirements table.

```markdown
## Derived Requirement Policy

Derived requirements are allowed when the FPGA architecture must introduce implementation
requirements not stated literally in the parent system documents.

Rules:

1. Derived requirements use ID format `FAD-DER-<CATEGORY>-NNN`.
2. Each must cite the parent requirement, ICD, ADR, or FAD section that forced it.
3. Each must appear in the RTM.
4. Each must map to at least one verification target.
5. Derived requirements cannot silently alter SFU functional ownership; boundary changes require an ADR.
```

### Change 4 — Add protocol assertions to ICD template

**Location:** ICD template, new §6 (renumber existing §6 Versioning to §7, §7 Open Issues to §8, §8 Change Log to §9).

```markdown
## 6. Protocol Assertions

| Assertion ID | Rule | Checked by |
|---|---|---|
| IF-SVA-001 | Payload and sideband remain stable while `valid && !ready`. | SVA / unit TB |
| IF-SVA-002 | `valid` is deasserted during reset and for the required reset recovery window. | SVA / unit TB |
| IF-SVA-003 | No unknown values on payload, sideband, or control when `valid` is high. | SVA / simulation |
| IF-SVA-004 | `last` occurs only at legal frame boundaries. | SVA / scoreboard |
| IF-SVA-005 | No sample is dropped or reordered under legal backpressure. | scoreboard |
```

Assertions are protocol-specific. The template carries generic AXI-Stream-like assertions; each instantiated ICD adjusts or extends the set.

### Change 5 — Add implementation evidence to MDS §12.3

**Location:** MDS §12, new subsection §12.3 after existing §12.2 (Post-synthesis actual). No renumbering of §13 (Open Issues) or §14 (Change Log).

Rationale: evidence is the post-implementation counterpart to the pre-synthesis estimate already in §12. They belong together. Adding a new top-level section (§13) would force renumbering across every existing MDS and template reference.

```markdown
### 12.3 Implementation evidence

| Check | Required before | Report / Evidence | Commit | Status |
|---|---|---|---|---|
| Lint | integration gate | `reports/<commit>/<module>/lint.log` | | |
| Unit simulation | integration gate | `reports/<commit>/<module>/unit_sim.xml` | | |
| Coverage | integration gate | `reports/<commit>/<module>/coverage.rpt` | | |
| OOC synthesis | integration gate | `reports/<commit>/<module>/synth_ooc.rpt` | | |
| OOC timing | integration gate | `reports/<commit>/<module>/timing_summary.rpt` | | |
| CDC | integration gate | `reports/<commit>/<module>/cdc.rpt` | | |
| Constraints check | integration gate | `reports/<commit>/<module>/check_timing.rpt` | | |
```

### Change 6 — Do NOT add `integration_ready` as a document lifecycle state

**Rationale:** `rtl_ready` is a *document completeness* state (is the spec sufficient for implementation to start?). "Integration-ready" is a *workflow evidence* state (has the RTL passed lint, sim, OOC, CDC?). These are fundamentally different tracks. Mixing them in the MDS `status` field conflates spec completeness with implementation status.

**Instead:**

- MDS lifecycle states remain: `draft | in_review | baselined | frozen | rtl_ready | not_rtl_ready | superseded`.
- Implementation status is tracked in MDS §12.3 (evidence table). Each row has a status column.
- `flow.md` and `signoff.md` define the "integration gate" — the workflow checkpoint evaluated by checking the evidence table.
- Optional machine-readable field in MDS frontmatter: `implementation_evidence: incomplete | complete`. Flipped when all §12.3 rows are filled. Lighter than a lifecycle state, no confusion about what `status` means.

### Change 7 — Soften Claude wording

**Current:** `rtl_ready — every RTL-ready criterion met; Claude Code can generate RTL from this + referenced ICDs`

**New:** `rtl_ready — the module specification is sufficient for RTL implementation to start, either manually or with Claude-assisted code generation. The MDS must be unambiguous enough that no implementation decision is left to the implementer.`

The second sentence preserves the original intent: the completeness bar is set by "can someone implement this without interpolation?", which is tool-agnostic.

### Change 8 — Add IP integration policy to `flow.md`

**Missing from original plan.** Required for SFU given Aurora, RFDC, clocking wizard, AXI infrastructure IP.

New section in `flow.md`:

```markdown
## IP Integration Policy

### IP source management
- Vendor IP configurations (`.xci` files) are versioned in `prj/ip/<ip_name>/`.
- IP is generated (output products) as part of the build flow, not committed.
- IP version is pinned to the Vivado release in the tool baseline (§3).

### IP upgrade policy
- IP version upgrade requires an ADR if it changes the IP interface, latency, resource usage, or simulation model behavior.
- Minor patch upgrades within the same Vivado release: no ADR, but regression required.

### IP-generated constraints
- IP-generated XDC files are not manually edited.
- Vivado flow ordering ensures IP constraints load before project constraints.
- Any project constraint that interacts with IP-internal paths must be documented in `05_timing_exceptions.xdc` with proof.

### Simulation models
- Vendor-compiled simulation models used for unit simulation.
- Behavioral models (not post-synthesis) used for OOC synthesis.
- Simulation model compilation is part of `make sim_unit` / `make sim_all`.

### Verification boundary
- Vendor IP internals are `not_applicable` for bit-exactness.
- The wrapper module (MDS reuse type = `wrap`) owns protocol and integration verification.
- Unit TB for a wrapper module verifies: correct parameterization, port connectivity, protocol compliance at wrapper boundaries, reset behavior.
```

### Change 9 — Split constraints and CDC methodology between policy docs and flow gates

**Problem in original plan:** `flow.md` §9 (constraints) and §10 (CDC) contain both *policy* (file structure, naming, approved patterns) and *gates* (required reports, pass/fail criteria). Policy should live closer to the artifacts it governs; gates belong in the flow.

**Split:**

| Content | Location |
|---|---|
| XDC file structure and naming conventions | `constraints/README.md` |
| XDC ownership rules per file | `constraints/README.md` |
| Approved/forbidden CDC patterns | FAD §4.5 (CDC inventory section, extended) |
| Constraints sign-off gate (reports, pass/fail) | `signoff.md` §6 |
| CDC sign-off gate (reports, waivers, pass/fail) | `signoff.md` §7 |
| Timing exception waiver policy | `flow.md` waiver policy section |
| CDC waiver format and storage | `flow.md` waiver policy section |
| `make cdc` / `make timing` commands | `flow.md` build entrypoints |

This avoids having two places where XDC organization is defined.

### Change 10 — Consolidate Claude-assisted usage to one location

**Problem:** Original plan has Claude sections in both `flow.md` §15 and `signoff.md` §14. Redundant.

**Fix:** Keep Claude-assisted usage in `flow.md` only. In `signoff.md`, replace with a one-line cross-reference: "See `flow.md` §15 for Claude-assisted workflow policy."

### Change 11 — Cross-reference flow gates to sign-off criteria

**Problem:** With `flow.md` and `signoff.md` as separate documents, a reader at a flow gate needs to find the corresponding sign-off criteria.

**Fix:** Each gate section in `flow.md` ends with a line: "Sign-off criteria: see `signoff.md` §N." Each sign-off section in `signoff.md` starts with: "Gate command: see `flow.md` §N."

### Change 12 — Add `constraints/README.md`

New file defining XDC organization. Content extracted from original plan's `flow.md` §9.1–§9.2:

```markdown
# Constraints

## XDC file structure

constraints/
├── 00_clocks.xdc
├── 01_generated_clocks.xdc
├── 02_resets.xdc
├── 03_io.xdc
├── 04_clock_groups.xdc
├── 05_timing_exceptions.xdc
├── 06_physical.xdc
├── 07_debug.xdc
└── ooc/
    └── <module_name>.xdc

## Ownership rules

- Clock definitions: `00_clocks.xdc` only.
- Generated clocks: `01_generated_clocks.xdc` only.
- IO delays: `03_io.xdc` only.
- Clock-domain relationships: `04_clock_groups.xdc` only.
- False paths and multicycle paths: `05_timing_exceptions.xdc` only.
- Physical constraints: `06_physical.xdc` only.
- Debug core constraints: `07_debug.xdc` only.
- IP-generated constraints are not manually edited.

## Timing exception discipline

Every false path or multicycle path entry must include:
- reason (inline comment)
- owner
- source/destination objects
- proof reference (module spec or ADR section)
- review status

Timing exceptions without documented proof are not allowed in baselined builds.
```

### Change 13 — Adopt module spec / micro-architecture split (ADR-0005)

**Context:** The project has separate architect and RTL designer roles. The designer has an existing LLM-assisted RTL flow (Claude Code) built around a black-box spec template. The current MDS template prescribes both external contract and internal micro-architecture, which removes designer implementation freedom and doesn't match the designer's workflow.

**Decision:** Adopt a two-level split:

| Document | Owner | Scope | Granularity |
|---|---|---|---|
| **Module spec** (`spec.md`) | Architect | External contract: ports, interfaces, clock domains, registers, operation, performance, errors, test hooks, system constraints | Functional block (~8–15 per project) |
| **Micro-architecture** (`uarch.md`) | RTL Designer | Internal implementation: FSMs, pipeline, FIFOs, sub-blocks, memory mapping | .sv files within a functional block |

**Terminology changes:**

| Before | After |
|---|---|
| MDS (Module Design Spec) | Module spec (`spec.md`) |
| `rtl_ready` (spec complete for junior engineer / Claude Code) | `design_ready` (spec complete for senior designer to start) |
| G4 MDS sign-off | G4a Spec sign-off (architect) + G4b uArch/RTL sign-off (designer) |
| `arch/modules/` | `arch/modules/` (unchanged) |

**Spec template:** Adopt the RTL designer's existing template format, augmented with framework conventions (YAML frontmatter, placeholder markers, citations, system-level constraints section, reference model pointer). Template adaptation is a separate task.

**FAD §6 module inventory:** Shifts from .sv-level (~31 entries) to functional-block level (~8–15 entries). The designer further decomposes each block into .sv files as part of the uarch.

**Reference model ownership:** DSP/numerical reference models are authored by the architect as part of the spec — they define the acceptance criterion. The designer implements unit TBs that correlate RTL against the architect-provided reference model.

**Derived requirements:** Flow into module specs as system-level constraints (latency budget, CDC mechanism, resource ceiling, fixed-point at boundaries). Traced in the RTM.

**Note:** The exact spec template is not yet frozen — agreement with the RTL designer on final template format is pending (ADR-0005). The methodology documents use the two-level split terminology; the `arch/modules/_template.md` will be finalized when ADR-0005 is accepted.

---

## Implementation sequence

Ordered by dependency — each step enables the next.

| Step | Change | Effort | Dependency |
|---|---|---|---|
| 1 | Rename FAD §12; soften `rtl_ready` wording (Changes 1, 7) | Low | None |
| 2 | Add architecture invariants as FAD §1.5 (Change 2) | Low | None |
| 3 | Add derived requirements policy to RTM (Change 3) | Low | None |
| 4 | Add protocol assertions to ICD template (Change 4) | Low | None |
| 5 | Add implementation evidence to MDS §12.3 (Change 5) | Low | None |
| 6 | Update repo layout — add `docs/methodology/`, `tb/`, `waivers/`, `reports/`; create `constraints/README.md` (Changes 9, 12, 13) | Medium | None |
| 7 | Create `flow.md` with IP integration policy, split constraints/CDC (Changes 8, 9, 10, 11) | High | Step 6 |
| 8 | Create `signoff.md` with cross-references to flow gates, G4a/G4b split (Changes 10, 11, 13) | Medium | Step 7 |
| 9 | Update `arch/README.md` — execution layer reference, updated lifecycle, spec/uarch terminology (Changes 6, 13) | Low | Steps 7, 8 |
| 10 | Replace `arch/modules/_template.md` with adapted spec template (Change 13) | Medium | ADR-0005 accepted |
| 11 | Defer automation (`make doc_check`, etc.) until templates stabilize | — | Steps 1–10 |

---

## Files touched

```text
arch/README.md                  — execution layer reference, lifecycle, spec/uarch terminology
arch/fad/_template.md           — §1.5 invariants, §12 rename + verification matrix, §6 inventory at block level
arch/modules/_template.md       — replaced: adapted from designer's spec template + framework conventions
arch/icd/_template.md           — §6 protocol assertions (renumber §6→§7, §7→§8, §8→§9)
arch/rtm.md                     — derived requirement policy
constraints/README.md           — new: XDC structure, ownership, exception discipline
docs/methodology/flow.md        — new: tool flow, gates, CI, IP policy, Claude usage, waivers
docs/methodology/signoff.md     — new: sign-off criteria per level, G4a/G4b split, cross-refs to flow
```

---

## What was changed from the original plan

| Original plan item | Disposition | Rationale |
|---|---|---|
| `integration_ready` as MDS lifecycle state | **Rejected** — use evidence table + workflow gate | Conflates document completeness with implementation status |
| MDS §13 Execution Evidence (new section) | **Modified** — MDS §12.3 instead | Avoids renumbering §13/§14; evidence belongs with resource/latency data |
| Architecture invariants after FAD §3 | **Modified** — FAD §1.5 | Invariants constrain the whole design; visible early |
| Claude section in both flow.md and signoff.md | **Trimmed** — flow.md only | Eliminate redundancy |
| Constraints/CDC methodology fully in flow.md | **Split** — policy in constraints/README.md and FAD §4.5; gates in signoff.md | Avoid defining XDC structure in two places |
| IP integration policy | **Added** — missing from original plan | Required for SFU (Aurora, RFDC, clocking, AXI IP) |
| `constraints/README.md` | **Added** — missing from original plan | XDC structure and ownership rules need a home |
| Repo layout update | **Added** — missing from original plan | `tb/`, `reports/`, `waivers/`, `docs/methodology/` need to be in the layout |
| Cross-references between flow and signoff | **Added** | Separate docs need explicit linkage at each gate |
| MDS as full blueprint | **Replaced** — module spec (black-box contract) + uarch (designer-owned) | Designer has existing flow; architect prescribing internals blocks designer freedom and creates bottleneck |
| G4 as single gate | **Split** — G4a (spec, architect) + G4b (uarch/RTL, designer) | Different owners, different criteria, different timing |
| `rtl_ready` as architect gate | **Replaced** — `design_ready` at spec level | Spec completeness ≠ RTL readiness; designer adds uarch to reach RTL-ready |
