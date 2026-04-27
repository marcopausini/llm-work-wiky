---
id: 2026-04-26--bpms-sfu-architecture--project-methodology-and-spec-uarch-split
date: 2026-04-26
project: bpms-sfu-architecture
type: decision
status: ongoing
topics: [methodology, workflow, spec-uarch-split, llm-assisted, architect-role, rtl-designer-role, ug949, review-gates]
source_chat: claude-opus-4.7
---

# FPGA project methodology established; module spec / micro-architecture split adopted

## Context

Extending the SFU FPGA architecture work from a documentation-only framework
(`fpga-arch-framework.md`) into a complete project methodology covering the full
V-model from architecture through hardware bring-up. Driven by needing to align
with the RTL designer's existing LLM-assisted flow (Claude Code), and by the
system architect asking for a delivery date.

Inputs reviewed: `fpga-prj-best-methodology.md` (UG949 summary),
`fpga-arch-framework-review.md` (ChatGPT review of current framework),
`fpga-arch-framework-revision-plan.md` (ChatGPT revision proposal),
`fpga-arch-design-review-workflow.md` (LLM role-split proposal).

## Key findings

### Methodology framework

- The 5-doc framework (FAD / Module Spec / ICD / ADR / RTM) is correct as a
  documentation system but missing an execution layer (tool flow, CI gates,
  sign-off criteria, IP integration policy).
- Two-layer approach adopted: keep `arch/` framework for specifications;
  add `docs/methodology/` for execution and sign-off.
- The architect's deliverable scope was too broad in the original framework —
  it conflated specification with micro-architecture.

### Module Spec / micro-architecture split

- Architect delivers Module Spec (MS): black-box external contract — ports,
  interfaces, clock domains, registers, operation, performance, errors,
  test hooks, system constraints (latency budget, CDC mechanism, resource
  ceiling, fixed-point at boundaries), reference model pointer for DSP/numerical
  modules.
- RTL designer delivers Micro-Architecture (uArch): internal implementation —
  FSMs, pipeline, FIFOs, sub-blocks, .sv decomposition.
- Granularity: architect specifies at functional-block level (~10 blocks for SFU);
  designer decomposes each block into multiple .sv files as part of uArch.
- One file per block in `arch/modules/<module_name>.md` (flat, no folder per
  module). Document type is "Module Spec" (acronym MS); filename uses
  module name only.
- `uarch.md` is NOT an architect deliverable. It lives alongside the RTL
  in the designer's flow.

### Lifecycle and review gates

- `rtl_ready` (architect-side) replaced by `design_ready` — sufficient for senior
  designer to start, not "junior engineer / Claude Code can implement directly."
- G4 split into G4a (Spec sign-off, architect) + G4b (uArch/RTL sign-off, designer).
- Five review gates total: G1 Orient (FAD §1–§6), G2 Contracts (FAD §7–§8 + ICDs),
  G3 Budgets (FAD §9), G4a Spec sign-off, G4b uArch/RTL sign-off, G5 Handoff to DV.

### Reference model ownership

- DSP/numerical reference models (Python bit-exact) authored by architect —
  they define the acceptance criterion, part of the specification.
- Unit testbenches authored by RTL designer — they implement the verification
  scenarios that the architect specified in the MS test plan hooks.

### LLM-assisted workflow

- Three roles for architecture phase: `fpga_arch` (Claude.ai, drafter),
  `fpga_arch_reviewer` (ChatGPT, adversarial reviewer), `repo_operator`
  (Claude Code, repo-aware executor).
- Pipeline: Draft → Review → Decide → Apply → Check.
- Anti-laundering discipline: every factual claim cited; every assumption
  marked with expiry trigger; every decision in an ADR.
- Codex role deferred until RTL phase begins.
- Role definitions live in tool config (Claude Project instructions, ChatGPT
  custom instructions), not in the design repo.

### Repo layout decisions

- Folder name `arch/modules/` kept (not renamed to `arch/specs/`).
- New folders added to layout: `docs/methodology/`, `tb/`, `reports/`, `waivers/`.
- `tb/` (unit TBs) lives in design repo; integration/UVM TBs live in
  `bpms-sfu-fpga-verif`.
- Reference models (`model/`) live in design repo — they are part of the
  specification.

### Execution flow ownership

- Architect authors `execution-flow.md` initially (defines gates and policies);
  designer fills in tool-specific Tcl/make targets as implementation proceeds.
- I/O plan and top-level port list are architect deliverables (constrained by
  RFX-8440A board); `sfu_top.sv` itself is designer-implemented.

## Decisions

- Adopt the spec/uArch split per Gastón's existing flow. Architect delivers
  external contract; designer owns internal implementation. Recorded as
  pending ADR-0005.
- Adopt Gastón's `spec.md` template as the basis for `arch/modules/_template.md`,
  augmented with framework conventions (frontmatter, markers, citations,
  system constraints section, refmodel pointer). Pending Gastón confirmation.
- Reference models for DSP/numerical modules are architect-authored.
- Three-document methodology layer in `docs/methodology/`:
  `fpga-project-methodology.md` (top-level), `architect-workflow.md`,
  `rtl-design-workflow.md`. Plus `execution-flow.md` and `signoff-criteria.md`.
- Phase 3 ("RTL implementation") includes uArch authoring, RTL coding, unit
  TBs, refmodel correlation, lint, simulation, and OOC synthesis.
- Folder rename `arch/modules/ → arch/specs/` rejected. Keep current name.

## Open questions

- Gastón's confirmation on the exact `spec.md` template — pending review of
  the two-level split proposal.
- Functional-block partitioning for SFU (~10 blocks) is an output of
  FAD §2 and §6 update, reviewed at G1. Not yet committed.
- Whether `fpga-arch-framework.md` continues to exist after revision plan
  execution, or merges into `arch/README.md`. Leaning toward: after revision
  plan, framework content lives in `arch/README.md`; the working file
  `fpga-arch-framework.md` is superseded.
- Source-doc open issues (OI-001, OI-S07, OI-S08) remain external blockers
  for specs that depend on them.

## Action items

- [ ] Apply revision plan v2 changes 1–9 (template patches, README updates)
- [ ] Get Gastón's confirmation on spec.md template (critical path item)
- [ ] Write and accept ADR-0005 (spec/uArch split)
- [ ] Replace `arch/modules/_template.md` with adapted spec template
- [ ] Create `constraints/README.md`
- [ ] Author `execution-flow.md` content
- [ ] Author `signoff-criteria.md` content
- [ ] Update top-level repo README to reference `docs/methodology/fpga-project-methodology.md`
- [ ] Update FAD §2 block diagram and §6 module inventory to spec-level granularity (~10 functional blocks)
- [ ] Define I/O plan and top-level port list (constrained by RFX-8440A board)
- [ ] Continue FAD §3–§5 authoring (dataflow, clocking/CDC, memory)
- [ ] Draft pilot module spec end-to-end with Gastón's flow to validate template
- [ ] Update Claude Project instructions with spec/uArch terminology and role definitions
- [ ] Update CLAUDE.md in repo to implement repo_operator role
- [ ] Create ChatGPT project `fpga-arch-reviewer` with instructions and context files
- [ ] Create presentation for next meeting covering full project methodology
- [ ] Update state.md (this checkpoint, separate task)

## Estimation given to system architect

- Milestone 1 (FAD baselined + ICDs frozen + ADR-0005 + pilot spec): 2 weeks
- Milestone 2 (all module specs at design_ready, RTM populated): 4 weeks after M1
- Total: 6 weeks, contingent on (a) Gastón's spec template agreement in week 1,
  (b) source-doc open issues being assigned owners, (c) no major architectural
  surprises in FAD §3–§5.
- Estimates assume LLM-assisted pipeline. Manual workflow would be ~10–12 weeks.
- LLM speedup applies to drafting, consistency checks, scaffolding, format work.
  Does NOT speed up engineering thinking, ambiguity resolution, or stakeholder
  alignment.

## Artefacts produced

- `projects/bpms-sfu-architecture/methodology/fpga-project-methodology.md` — top-level project methodology entry point: roles, phases, document types, repo layout, conventions, gates
- `projects/bpms-sfu-architecture/methodology/architect-workflow.md` — detailed FPGA architect workflow with LLM-assisted draft/review/decide/apply/check pipeline
- `projects/bpms-sfu-architecture/methodology/rtl-design-workflow.md` — placeholder stub for RTL designer workflow (to be authored by Gastón)
- `projects/bpms-sfu-architecture/methodology/execution-flow.md` — placeholder stub for tool flow, CI gates, IP integration policy
- `projects/bpms-sfu-architecture/methodology/signoff-criteria.md` — placeholder stub for sign-off criteria per gate level
- `projects/bpms-sfu-architecture/planning/fpga-arch-framework-revision-plan-v2.md` — 13 changes to extend framework into complete methodology, with implementation sequence
- `projects/bpms-sfu-architecture/planning/fpga-methodology-tasks.md` — 19-task list with critical path and checkboxes

## Re-seed block

GOAL: Continue the SFU FPGA architecture work — execute the revision plan v2 changes, finalize spec template with Gastón, update FAD §2/§6 to spec-level granularity, draft pilot module spec.

CONSTRAINTS:
- Target device: BittWare RFX-8440A (AMD XCZU43DR-2FFVE1156E)
- Source of truth: BPMS-1.0-SFU-001 v1.6 (primary), BPMS-1.0-ARCH-001 v2.4 (secondary)
- 5-doc framework (FAD/MS/ICD/ADR/RTM); flat `arch/modules/<module_name>.md`
- Architect delivers black-box specs, not micro-architecture
- Reference models (Python) for DSP modules are architect-authored
- LLM pipeline: Claude.ai drafts → ChatGPT reviews → engineer decides → Claude Code applies
- Author name: Marco Pausini

PRIOR CONCLUSIONS:
- Module Spec / uArch split adopted (pending ADR-0005)
- `design_ready` replaces `rtl_ready` for spec-level criterion
- G4 split into G4a (architect) + G4b (designer)
- Folder name `arch/modules/` kept; one flat `<module_name>.md` per block
- Reference model ownership: architect for DSP/numerical, designer for protocol
- Three methodology documents complete: `fpga-project-methodology.md`, `architect-workflow.md`, plus three placeholder stubs
- Estimation: 6 weeks total (2 to M1, 4 to M2) with LLM-assisted pipeline

CURRENT QUESTION: Confirm spec template with Gastón, then update FAD §2/§6 to spec-level granularity and draft pilot module spec to validate the template end-to-end.
