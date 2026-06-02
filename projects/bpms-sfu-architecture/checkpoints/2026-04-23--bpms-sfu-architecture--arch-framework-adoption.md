---
id: 2026-04-23--bpms-sfu-architecture--arch-framework-adoption
date: 2026-04-23
project: bpms-sfu-architecture
type: decision
status: ongoing
topics: [arch-framework, adr, documentation, mds, icd, rtm, fad, arc42, madrs, ug949, repo-layout]
source_chat: claude-opus-4.7
---

# SFU FPGA documentation framework — 5-doc Markdown, ADR-0001 proposed; repo `bpms-sfu-fpga-design` with flat layout

## Context

Bootstrapping FPGA documentation for BPMS 1.0 SFU on BittWare RFX-8440A (XCZU43DR). Deliverable pipeline: `system docs → .md FPGA-architecture docs → .md block specs → Claude Code → SystemVerilog RTL`. Two proposals evaluated — a baseline 5-doc structure (FAD/MDS/ICD/ADR/RTM) and an extended 15-doc structure. Reviewed against RTL-ready criteria and against NASA GSFC / arc42 / MADRs / UG949 / Wipro structural patterns from the FPGA Architect Resource Guide. After initial scaffold, team naming convention (`bpms-<device>-fpga-<design|verif>`) surfaced and the repo was renamed and restructured before the first commit.

## Key findings

- 5-doc structure wins on coordination cost for a solo architect. 15 files amplifies cross-doc drift (clock domain list defined in 3 places), inflates Claude Code context assembly, duplicates FAD §1–§6 content in an `architecture_overview.md`.
- Every valuable piece from the extended proposal absorbed as depth into the 5-doc structure: functional-boundary ownership table → FAD §1.2; minimum-telemetry contract → FAD §10.1 + MDS §9.5; arc42 key-decisions view → FAD §1.4 table; UG949 review flow → 5-gate baselining table (G1 Orient → G5 Handoff to DV) in `arch/README.md`; MADRs → ADR template with `supersedes`/`superseded_by` lineage.
- YAML frontmatter added to every template. `status`, `rtl_ready_blocking: [...]`, `bit_exactness: {bit_exact|ulp_bounded|not_applicable}` are the three machine-checkable fields that matter most — enable a pre-commit hook in `scripts/` to block merges when RTL-ready claims don't hold.
- Placeholder markers (`[TBD:]` / `[STUB:]` / `[ASSUMPTION:]` / `[INFERRED:]`) are greppable and preserve the specified/inferred/proposed split that `state.md` demands.
- Repo naming settled on team convention: `bpms-sfu-fpga-design` and `bpms-sfu-fpga-verif` (planned) — applies symmetrically to DCU. Earlier proposal `bpms-sfu-fpga/` with `arch/ code/ workflow/` retired; the `-design` suffix makes "design" the whole repo, so an interior `code/` folder is redundant.
- **Top-level layout is flat and organised by artefact type**, not by V-model phase: `arch/ rtl/ model/ constraints/ prj/ scripts/ docs/`. Rationale: artefacts cross phases (MDS is authored in arch phase, referenced during implementation, cited during verif); tools and humans search by artefact type. V-model review gates (G1–G5) live in `arch/README.md`, not in folder names.
- Reference models (`model/`) live in the **design repo**, not verif. Refmodel is part of the specification, authored before RTL, referenced by MDS §3.3. Verif repo consumes `model/` via fixed-revision pin from the design repo.
- Framework doc renamed `fpga_design_templates.md` → `fpga-arch-framework.md` — "framework" reflects that it contains methodology + conventions + folder layout + templates, not just templates.
- Project FAD skeleton seeded with content derivable from source docs: §1.2 functional boundary (full in/out-scope tables), §1.4 key-decisions seed, §1.7 target device, §4.1 clock inventory, §4.5 CDC inventory seed, §8.3 growth/rounding/saturation policy, §10.1 min telemetry, §11 scheduled-apply and autonomous-update mechanisms. Rest is explicit `[STUB:]` / `[TBD:]`.
- FAD skeleton held back from first commit — first commit stages templates + ADR-0001 only. Keeps "framework adopted" and "first project FAD drafted" as separate reviewable events.

## Decisions

- Adopt 5-doc framework (FAD, MDS, ICD, ADR, RTM). ADR-0001 drafted with status `proposed`. Flips to `accepted` after the first MDS validates the template end-to-end.
- Repo: `bpms-sfu-fpga-design/` (design-side, this repo) and `bpms-sfu-fpga-verif/` (verif-side, planned). Symmetric to DCU.
- Top-level folders in `bpms-sfu-fpga-design/`: `arch/`, `rtl/`, `model/`, `constraints/`, `prj/`, `scripts/`, `docs/`. Flat, by artefact type.
- Framework doc name: `fpga-arch-framework.md` (outside the repo, working artifact).

## Open questions

- Top-level block diagram (FAD §2) — blocks module inventory §6 stabilisation.
- ADR-0002: fixed-point notation (`Q(I.F)` vs `s(W,F)`) — blocks FAD §8 and every DSP MDS. Next decision.
- ADR: streaming-bus convention (AXI-Stream candidate) — blocks FAD §7 and every inter-module ICD.
- ADR: unit-TB stack (cocotb candidate) and refmodel language (Python candidate) — one combined or two.
- Vivado version pinning (FAD §1.7) — must be pinned before `prj/build.tcl` is authored.

## Action items

- [ ] Commit templates + ADR-0001 (proposed) to new `bpms-sfu-fpga-design` repo.
- [ ] Re-upload updated `state.md` to the Claude Project knowledge base.
- [ ] Flip ADR-0001 to `accepted` once first MDS closes.
- [ ] Implement frontmatter-lint pre-commit hook in `scripts/` after first MDS exercises the schema.


## Artefacts produced

Repo-targeted (first commit — templates, folder READMEs, ADR-0001):
- `bpms-sfu-fpga-design/README.md` — top-level README (design/verif split, flat layout)
- `bpms-sfu-fpga-design/arch/README.md` — framework spec (layout, lifecycle, markers, citation discipline, review gates)
- `bpms-sfu-fpga-design/arch/fad/_template.md` — FAD template
- `bpms-sfu-fpga-design/arch/modules/_template.md` — MDS template with RTL-ready self-check
- `bpms-sfu-fpga-design/arch/icd/_template.md` — ICD template
- `bpms-sfu-fpga-design/arch/adr/_template.md` — ADR template (MADR-shaped)
- `bpms-sfu-fpga-design/arch/adr/0001-adopt-5-doc-markdown-framework.md` — framework-adoption ADR, status `proposed`
- `bpms-sfu-fpga-design/arch/rtm.md` — RTM skeleton (living doc)
- `bpms-sfu-fpga-design/rtl/README.md` — RTL layout conventions, Claude Code prompt recipe
- `bpms-sfu-fpga-design/model/README.md` — refmodel layout and bit-exactness policy
- `bpms-sfu-fpga-design/constraints/README.md` — top-level XDC scope
- `bpms-sfu-fpga-design/prj/README.md` — Vivado project policy
- `bpms-sfu-fpga-design/scripts/README.md` — automation, hooks, codegen
- `bpms-sfu-fpga-design/docs/README.md` — rendered docs and diagrams

Repo-targeted (held back from first commit):
- `bpms-sfu-fpga-design/arch/fad/FAD.md` — project FAD skeleton, seeded with functional boundary and other derivable content

Outside the repo:
- `fpga-arch-framework.md` — framework definition, working artifact from the merge review
- `state.md` (updated) — for re-upload to the Claude Project KB


## Re-seed block

GOAL: Read the SFU architecture document end-to-end and draft the SFU FPGA top-level block diagram for FAD §2. This is the first pass through the source documents — no prior analysis to build on. The output is the diagram(s), a module naming convention, and a preliminary RTL module list to seed FAD §6 Module Inventory.

CONSTRAINTS:

- Target: BittWare RFX-8440A (AMD Zynq UltraScale+ RFSoC XCZU43DR-2FFVE1156E, speed grade -2).
- Framework is 5-doc Markdown (FAD/MDS/ICD/ADR/RTM) — ADR-0001 `proposed`. Do not propose alternative document structures.
- Repo is `bpms-sfu-fpga-design/` with flat layout: `arch/ rtl/ model/ constraints/ prj/ scripts/ docs/`. Do not propose alternative repo layouts. Verif lives in companion `bpms-sfu-fpga-verif/`.
- Diagram source: Mermaid inline preferred for diffability; `.drawio` acceptable if Mermaid can't express it.
- Every block in the diagram must correspond to an RTL module that will appear in FAD §6 Module Inventory. No conceptual blocks that don't map to an MDS.
- Every named block traces to one of: (a) an SFU-001 / ARCH-001 section that describes the function it implements (cite inline), (b) `[INFERRED from <§>]` — a decomposition or aggregation derived from a cited functional description, or (c) `[ASSUMPTION: <text>, <expiry trigger>]` — an implementation-driven block with no direct functional source, justified inline.
- Citation discipline: every factual claim cites SFU-001 §x or ARCH-001 §y; `[INFERRED]` / `[ASSUMPTION]` / `[TBD]` / `[STUB]` markers are mandatory for anything not literally stated.
- `state.md` is canonical; contradictions with chat or instructions are resolved in favour of `state.md`.

PRIOR CONCLUSIONS (process only — no architectural analysis has been performed):

- 5-doc framework chosen (FAD / MDS / ICD / ADR / RTM). ADR-0001 `proposed`.
- Repo name `bpms-sfu-fpga-design` and top-level layout (`arch/ rtl/ model/ constraints/ prj/ scripts/ docs/`) are settled.
- Reference models live in the design repo under `model/`.
- FAD.md exists as a skeleton with empty sections. No sections have been populated from the source documents yet.

CURRENT QUESTION: Read `BPMS_1.0_SFU_Architecture_v1.6.docx` (primary) and `BPMS_1.0_Architecture_Document_v2.4.docx` (secondary, only when SFU doc references it) from scratch. Extract the SFU signal chain, identify every functional block, and produce the top-level block diagram for FAD §2. Before drawing anything:

1. Define a module naming convention. Get approval.
2. Identify every functional block implied by SFU-001, cite the source section, and present the block list for approval before diagramming.
3. Ask clarifying questions wherever the source documents leave structural choices open or multiple decompositions are defensible.
