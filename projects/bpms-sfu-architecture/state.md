# bpms-sfu-architecture — canonical state

**Last updated:** 2026-04-26
**Mirrored to Claude Project:** BPMS SFU Architecture

## Current phase

Project methodology established. Documentation framework extended from
specification-only to complete project methodology covering V-model from
architecture through hardware bring-up. Module Spec / Micro-Architecture
split adopted (pending ADR-0005 and Gastón's confirmation on spec template).

**Active milestone — Milestone 1 (target: 2 weeks).** Outputs:
FAD §1–§12 baselined, core ICDs frozen, ADR-0005 accepted, pilot module
spec drafted end-to-end with Gastón's RTL flow.

**Next phase — Milestone 2 (target: 4 weeks after M1).** Outputs:
all module specs at `design_ready`, RTM populated, architecture done.

## What exists

### Project documentation
- Project instructions: role, conventions, design-ready criteria, spec/uArch split, LLM-assisted workflow, response format — refreshed 2026-04-27 (author Marco Pausini)
- `state.md` — this file, canonical state
- Companion repo `bpms-sfu-fpga-verif/` (planned, not yet created)

### Framework documents — status

The framework split into a *spec layer* (`arch/README.md`) and an *execution layer* (`docs/methodology/`) is now realized in the design repo. Working artifacts that were used to drive the split are kept here as historical inputs.

In-repo (final, authoritative):
- `bpms-sfu-fpga-design/arch/README.md` — **superseded `fpga-arch-framework.md` as of 2026-04-27**. Single source of truth for spec layer (document types, layout, lifecycle, baselining).
- `bpms-sfu-fpga-design/docs/methodology/01-fpga-project-methodology.md` — top-level methodology entry point (roles, phases, repo layout).
- `bpms-sfu-fpga-design/docs/methodology/02-architect-workflow.md` — architect role, drafting, LLM-assisted review, gates G1–G4a.
- `bpms-sfu-fpga-design/docs/methodology/03-rtl-design-workflow.md` — **placeholder stub**, to be authored by Gastón (RTL designer role, gates G4b and integration).
- `bpms-sfu-fpga-design/docs/methodology/04-execution-flow.md` — tool baseline, `make` targets, per-stage gate commands, IP / waiver / archival policy.
- `bpms-sfu-fpga-design/docs/methodology/05-signoff-criteria.md` — gate-by-gate sign-off pass/fail criteria.

Working artifacts (outside repo, historical):
- `fpga-arch-framework.md` — original framework definition. **Superseded by `bpms-sfu-fpga-design/arch/README.md` (final 2026-04-27).** Retained for chat-history continuity only.
- `fpga-prj-best-methodology.md` — UG949 summary; input to execution layer.
- `fpga-arch-framework-review.md` — ChatGPT review of framework gaps; input to revision plan.
- `fpga-arch-framework-revision-plan.md` — superseded by v2.
- `fpga-arch-framework-revision-plan-v2.md` — 13-change plan, **executed 2026-04-26 / 2026-04-27**.
- `fpga-arch-design-review-workflow.md` — LLM role-split proposal; realized in `02-architect-workflow.md`.
- `fpga-methodology-tasks.md` — live 19-task list with critical path; tracks remaining methodology work.

### Repo scaffold (`bpms-sfu-fpga-design/`)
Flat top-level layout:
- `arch/` — specifications (FAD, module specs, ICD, ADR, RTM)
- `rtl/` — SystemVerilog sources, per-module subdirectories
- `model/` — Python bit-exact reference models (architect-authored for DSP)
- `tb/` — unit testbenches (designer-authored)
- `constraints/` — top-level XDC + `README.md` (XDC structure, ownership)
- `prj/` — Vivado project Tcl, IP Integrator, `.xci` IP configurations
- `scripts/` — build, codegen, lint, pre-commit hooks, Claude prompt templates
- `docs/` — rendered docs, diagrams, slide decks
  - `docs/methodology/` — methodology and workflow documents (new)
- `reports/` — tool-generated evidence (gitignored or CI-archived)
- `waivers/` — lint, CDC, timing waivers (version-controlled)

### Architecture artifacts in `arch/`
- `arch/adr/0001-adopt-5-doc-markdown-framework.md` — status `proposed`
- `arch/fad/FAD.md` — populated through §2 (top-level block diagram, three views) and §6 (31-module inventory at .sv-level — **needs update to spec-level granularity ~10 functional blocks**). All other sections still skeleton or seeded. Awaiting review.
- `arch/fad/diagrams/` — three Mermaid diagrams (dl-datapath.md, ul-datapath.md, infrastructure.md)

## Load-bearing technical facts

### SFU ownership

- Owns: OBG ingest, lane alignment, bins selection and routing,
  filter-bank synthesis and analysis, band gain, band Doppler, RF
  interfacing, timing, management, debug.
- Does NOT own: per-beam functions — beam gain, beam Doppler, beam
  delay compensation. Those belong to DCU.
- Any proposal that crosses DCU/SFU boundary must be flagged explicitly.
- Authoritative ownership table lives in `arch/fad/FAD.md` §1.2 with boundary-crossing procedure.

### Scope boundary — PS software

- This repo (`bpms-sfu-fpga-design`) covers PL RTL only. PS software is out of scope.
- PS software includes: TLE/SGP4 ephemeris propagation, mgmt protocol stack (REST/SNMP/TBD per OI-S07), Ethernet GEM driver, NMS-side discovery and provisioning logic.
- PL-side blocks `mgmt` and `band_doppler` (which includes TLE compute) are documented as PL components with defined PS↔PL interfaces. The PS counterparts will live in a separate repo (TBD).
- PS-PL split details to be captured in ADR-0003 when authored.

### Module Spec / Micro-Architecture split (NEW 2026-04-26)

- Architect delivers Module Spec (MS): black-box external contract.
  - Filename: `arch/modules/<module_name>.md` (flat, one per functional block)
  - Acronym: MS
  - Lifecycle gate: G4a (`design_ready`)
  - Content: ports, interfaces, clock domains, registers, operation, performance, errors, test hooks, system constraints (latency budget, CDC mechanism, resource ceiling, fixed-point at boundaries), reference model pointer
- RTL designer delivers Micro-Architecture (uArch): internal implementation.
  - Filename: designer's choice (e.g., `rtl/<module>/uarch.md`)
  - Lifecycle gate: G4b (implementation evidence)
  - Content: FSMs, pipeline, FIFOs, sub-blocks, .sv decomposition
- Granularity: architect specifies at functional-block level (~10 blocks);
  designer decomposes into multiple .sv files per block.
- The two are distinct document types with different owners — both Markdown
  but different folders, different lifecycles, different content scope.

### Reference model ownership (NEW 2026-04-26)

- DSP/numerical reference models (Python bit-exact) authored by architect.
  Live in `model/` in design repo. Part of the specification — define
  acceptance criterion.
- Unit testbenches authored by RTL designer. They implement the verification
  scenarios defined by the architect in MS test plan hooks, and correlate
  RTL against architect-provided reference models.

### LLM-assisted workflow (NEW 2026-04-26)

- Three roles for architecture phase:
  - `fpga_arch` (drafter) — Claude.ai
  - `fpga_arch_reviewer` (reviewer) — ChatGPT or independent LLM
  - `repo_operator` (executor) — Claude Code
- Pipeline: Draft → Review → Decide → Apply → Check.
- Anti-laundering discipline: every claim cited; every assumption marked
  with expiry trigger; every decision in an ADR.
- Codex role deferred until RTL phase.
- Role definitions live in tool config (Project instructions, ChatGPT custom
  instructions, CLAUDE.md), not in the design repo.
- RTL designer LLM workflow (Claude Code) defined separately by Gastón
  in `rtl-design-workflow.md`.

### Source of truth

- Uploaded project documents and `state.md` are the source of truth.
- Primary: `BPMS_1.0_SFU_Architecture_v1.6.docx` (BPMS-1.0-SFU-001).
- Secondary: `BPMS_1.0_Architecture_Document_v2.4.docx` (BPMS-1.0-ARCH-001).
- When artifacts conflict, prefer more specific and more recent.
- Always separate: explicitly specified / inferred / proposed.
- Citation discipline: every spec claim cites document and section.
- If anything contradicts `state.md`, `state.md` wins.
- Placeholder markers: `[TBD: <reason>, <owner>]`, `[STUB: <blocker>]`,
  `[ASSUMPTION: <text>, <expiry trigger>]`, `[INFERRED from <source §>]`.

### Repo layout principles

- Flat at top level, organized by artefact type, not by lifecycle phase.
- Design and verification are separate repos (`-design` / `-verif` suffix).
- Reference models live in design repo (architect-authored part of spec).
- Unit testbenches live in design repo (designer-authored).
- Reports gitignored or CI-archived; waivers version-controlled.

## Architecture / design choices made

### From earlier work
- **ADR-0001 (proposed)** — adopt 5-doc Markdown framework (FAD / MS / ICD / ADR / RTM).
- **Repo split** — `bpms-sfu-fpga-design/` and `bpms-sfu-fpga-verif/` (planned).
- **Flat top-level layout** — `arch/ rtl/ model/ tb/ constraints/ prj/ scripts/ docs/ reports/ waivers/`.
- **Module naming convention** — no `sfu_` prefix (repo is SFU-scoped); `sfu_top` is sole exception. DL/UL suffix only when same function exists in both directions as separate RTL modules. Whitelist abbreviations: `obg`, `dl`, `ul`, `rx`, `tx`, `sel`, `synth`, `mgmt`, `if`, `rfdc`, `cdc`, `pps`, `rms`, `tle`, `cw`. Spell out everything else.
- **FAD §2 three-view organization** — DL datapath, UL datapath, infrastructure (mgmt + timing + shared). Clocks deferred to FAD §4 with their own diagram. Sub-band A/B handled inside each block.

### From this session (2026-04-26)
- **ADR-0005 (pending)** — Module Spec / Micro-Architecture split. Architect delivers black-box spec; designer owns internal implementation.
- **Spec template** — Gastón's existing `spec.md` template adopted as basis, augmented with framework conventions (frontmatter, markers, citations, system constraints section, refmodel pointer). Pending Gastón confirmation.
- **`design_ready` replaces `rtl_ready`** — spec-level lifecycle state, sufficient for senior designer to start.
- **G4 split into G4a + G4b** — G4a Spec sign-off (architect), G4b uArch/RTL sign-off (designer).
- **Folder rename rejected** — `arch/modules/` kept (not renamed to `arch/specs/`).
- **One flat file per block** — `arch/modules/<module_name>.md`. No folder per module unless future need arises.
- **MS acronym adopted** — Module Spec gets acronym MS for consistency with FAD/ICD/ADR/RTM.
- **Reference model ownership clarified** — architect authors DSP/numerical refmodels.
- **31-module inventory revised** — needs update to ~10 functional blocks at spec-level granularity. Block partitioning is an output of FAD §2/§6 update, reviewed at G1.
- **Methodology documents created** — `fpga-project-methodology.md`, `architect-workflow.md`, plus three placeholder stubs in `docs/methodology/`.
- **Revision plan v2** — 13 changes documented to extend framework into complete methodology.

## Open questions / decisions

- **Gastón's spec template confirmation** — critical path item, blocks ADR-0005 and template replacement.
- **ADR-0001 acceptance** — flips to `accepted` after first MS validates the template end-to-end.
- ~~**`fpga-arch-framework.md` future**~~ — **resolved 2026-04-27**: superseded by `bpms-sfu-fpga-design/arch/README.md` (final).
- **FAD §2 / §6 review** — needs update to spec-level granularity, then submit to fpga-arch-reviewer for G1 review.

## Active flags

These items emerged during FAD §2 authoring or this session. Each must land in FAD §13 (Open Issues), an ADR, or both.

| # | Flag | Triggers | Source |
|---|---|---|---|
| 1 | `cdc_fifo_dl` placement (before vs after `obg_phase_align`); `cdc_fifo_ul` placement (before vs after `bins_sel_ul`). SFU-001 does not explicitly state the crossing point. Recommended: DSP-clock side for bin-domain logic; CDC sits at the OBG boundary. | ADR-0002 (clock architecture) | Earlier |
| 2 | `band_doppler` (which includes TLE compute) — PS-PL split: PS does SGP4 propagation, PL does 1PPS-edge NCO load | ADR-0003 (PS-PL partitioning) | Earlier |
| 3 | `mgmt` PS-PL split — PS hosts GEM + protocol stack, PL hosts AXI bridge into reg bus | ADR-0003 | Earlier |
| 4 | PS software out of scope of `bpms-sfu-fpga-design` | state.md + ADR-0003 | Earlier |
| 5 | UTC apply engine centralized vs distributed shadow registers | MS-time decision; possible ADR-0004 | Earlier |
| 6 | NV-restore-on-POR FSM placement | MS-time decision | Earlier |
| 7 | `event_log` blocked on OI-S08 | not_design_ready until OI-S08 resolves | Earlier |
| 8 | `capture` partly blocked on OI-S08 | not_design_ready until OI-S08 resolves | Earlier |
| 9 | `spur_det` + `spectrum_monitor` may merge | not_design_ready until OI-S10 resolves | Earlier |
| 10 | `cw_beacon` injection mechanism | MS-time decision | Earlier |
| 11 | DL capture tap B (post `band_doppler_dl`) is `[ASSUMPTION]` pending OI-S08 | revisit at OI-S08 resolution | Earlier |
| 12 | UL capture tap D (post `bins_sel_ul`) is `[ASSUMPTION]` pending OI-S08 | revisit at OI-S08 resolution | Earlier |
| 13 | FAD §6 module inventory needs update from 31 .sv-level entries to ~10 functional blocks at spec-level granularity | FAD §2/§6 update | This session |
| 14 | Spec template for `arch/modules/_template.md` pending Gastón confirmation | ADR-0005 | This session |

## ADR pipeline (planned)

| ADR | Topic | Trigger |
|---|---|---|
| ADR-0001 | Adopt 5-doc Markdown framework | proposed |
| ADR-0002 | Clock architecture (CDC inventory, primary/secondary topology, FIFO placement) | Pending FAD §4 authoring |
| ADR-0003 | PS-PL partitioning (TLE compute, mgmt interface, PS scope boundary) | Pending FAD §11 authoring |
| ADR-0004 | UTC apply engine — centralized vs distributed | Deferred to first scheduled-apply MS |
| ADR-0005 | Module Spec / Micro-Architecture split | Pending — drafted but not yet committed |

## Estimation given to system architect

- **Milestone 1 — 2 weeks:** FAD §1–§12 baselined, core ICDs frozen, ADR-0005 accepted, pilot module spec drafted end-to-end with Gastón's flow.
- **Milestone 2 — 4 weeks after M1:** All module specs at `design_ready`, RTM populated, architecture officially done.
- **Total: 6 weeks**, contingent on:
  1. Gastón's spec template agreement in week 1
  2. Source-doc open issues being assigned owners
  3. No major architectural surprises in FAD §3–§5 authoring
- Estimate assumes LLM-assisted pipeline. Manual workflow would be ~10–12 weeks.

## Next concrete actions

Critical path: get Gastón's confirmation → ADR-0005 accepted → spec template
replacement → FAD §2/§6 update → pilot spec.

Parallel work:
- Apply revision plan v2 changes 1–9 (template patches)
- Create `constraints/README.md`
- Author `execution-flow.md` and `signoff-criteria.md` content
- Update Claude Project instructions
- Update CLAUDE.md to implement repo_operator
- Create ChatGPT project `fpga-arch-reviewer`
- Create presentation for next meeting

Full task list with checkboxes: `fpga-methodology-tasks.md`.

## Active references

- BPMS 1.0 System Architecture spec v2.4 (S. Kulik) — uploaded to project KB (title + revision only — do not paste content)
- BPMS 1.0 SFU Architecture v1.6 (S. Kulik) — uploaded to project KB
- `docs/methodology/01-fpga-project-methodology.md` — current authoritative methodology entry point
- `docs/methodology/02-architect-workflow.md` — drafter role + LLM pipeline definition
- `docs/methodology/03-rtl-design-workflow.md` — RTL designer workflow (stub, Gastón to author)
- `docs/methodology/04-execution-flow.md` — Vivado flow, CI gates, IP integration policy
- `docs/methodology/05-signoff-criteria.md` — G1–G5 / G4a-G4b sign-off semantics
- `bpms-sfu-fpga-design/arch/README.md` — spec-layer source of truth (final 2026-04-27); supersedes `fpga-arch-framework.md`
- `fpga-prj-best-methodology.md` — UG949 summary, reference for execution layer
- `fpga-methodology-tasks.md` — live 19-task list with critical path

## Recent changes

- 2026-04-27: framework finalization. `bpms-sfu-fpga-design/arch/README.md` finalized as the spec-layer source of truth, superseding the working artifact `fpga-arch-framework.md`. All five `docs/methodology/01–05-*.md` files mirrored into the wiki under `projects/bpms-sfu-architecture/methodology/` for KB upload. `04-execution-flow.md` and `05-signoff-criteria.md` realized from stubs into complete documents (only `03-rtl-design-workflow.md` remains a stub, awaiting Gastón). Project instructions and `state.md` refreshed to use the new `arch/README.md` + `01-…/05-…` references; `MDS → MS`, `RTL-ready → design-ready`, `MDS-time → MS-time` propagated everywhere. Spec/uArch ownership, review-gate (G1–G5 with G4a/G4b), and LLM-assisted-workflow sections added to project instructions. Author identity (Marco Pausini) recorded in active context.
- 2026-04-26: project methodology established. Module Spec / Micro-Architecture split adopted (ADR-0005 pending Gastón confirmation). Created `docs/methodology/` documents: `fpga-project-methodology.md` (complete), `architect-workflow.md` (complete), `rtl-design-workflow.md` (stub), `execution-flow.md` (stub), `signoff-criteria.md` (stub). Revision plan v2 documents 13 changes to extend framework into complete methodology. `design_ready` replaces `rtl_ready` for spec-level criterion. G4 split into G4a (architect) + G4b (designer). MS acronym adopted for Module Spec. Reference model ownership: architect authors DSP/numerical refmodels. FAD §6 inventory needs update from 31 .sv-level entries to ~10 functional blocks. Folder rename `arch/modules/ → arch/specs/` rejected. Estimation: 6 weeks total (M1 in 2 weeks, M2 in 4 weeks after).
- 2026-04-25: drafted FAD §2 (top-level block diagram, three views) and §6 (31-module inventory) from SFU-001 §4–§8, §11, §14–§15. Established module naming convention. Three Mermaid diagrams created. Defined PS software scope boundary. Identified 12 active flags and 3 ADR triggers.
- 2026-04-23 (late): restructured repo layout — `bpms-sfu-fpga/` → `bpms-sfu-fpga-design/`; dropped `code/` and `workflow/`; added flat top-level folders `rtl/`, `model/`, `constraints/`, `prj/`, `scripts/`, `docs/`. Updated path references.
- 2026-04-23 (early): drafted framework by merging baseline + extended template proposals; created repo scaffold; seeded FAD skeleton. Wrote ADR-0001 `proposed`. Framework filename settled as `fpga-arch-framework.md`.
- 2026-04-22: separated state.md from project instructions.
- 2026-04-21: initial state seeded from Project instructions.
