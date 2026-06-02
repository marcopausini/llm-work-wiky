---
id: 2026-04-28--bpms-sfu-architecture--spec-template-finalisation
date: 2026-04-28
project: bpms-sfu-architecture
type: decision
status: ongoing
topics: [methodology, spec-template, cdc-philosophy, waiver-policy, execution-flow, terminology]
source_chat: claude-sonnet-4-6
---

# Module spec template finalised; CDC philosophy locked; execution-flow cleaned

## Context

Continuation of the project-methodology work (see 2026-04-26 checkpoint).
Focus: finalise the `arch/modules/_template.md` for Gastón's review (Task 8/10),
lock the CDC-architecture philosophy, and sweep stale terminology from
`04-execution-flow.md`. Based on Gastón's `ip-specs.md` baseline and
`module-specs.md` (beam_injector filled example).

## Key findings

### Spec template

- Gastón's `ip-specs.md` is the correct baseline. His structure, comments,
  managed-block markers, YAML-register pattern, and WaveDrom sibling-file
  pattern are all load-bearing and must be preserved verbatim.
- Six architect-side additions are required and non-negotiable for the
  project framework: YAML frontmatter, Fixed-point format section,
  System-level constraints section, Acceptance criterion + Design-ready
  self-check sub-sections inside Test plan hooks, Implementation evidence
  section, and optional Operation sub-headers (commented by default).
- CDC mechanism per crossing was removed from the template after discussion.
  Rationale: FAD §4.5 holds the project-wide CDC policy (clock relationships,
  approved mechanisms, intentional crossings). Per-spec CDC mechanism adds
  duplicate work and a drift risk. Gastón's "Domain crosses" column in
  Clocks and resets is sufficient at the spec level.
- "Design-ready self-check" checkbox updated: was "required CDC mechanisms
  specified; every CDC mirrored in FAD §4.5" → now "domain crosses identified;
  every crossing recorded in FAD §4.5". Same enforcement, no mechanism detail
  in the spec.
- Two template versions produced: annotated (`_template.md`, 262 lines —
  for Gastón review, carries ARCH-ADDITION markers and delta-summary header)
  and clean (`_template_clean.md`, 214 lines — for repo commit, no scaffolding).

### CDC philosophy

- FAD §4.5 is architecture-level only: clock-domain relationships
  (synchronous-derived / mesochronous / asynchronous), approved-mechanism
  policy list, intentional architectural boundary crossings.
- FAD §4.5 is NOT a per-flop-pair inventory. Vivado `report_cdc` does
  the per-flop enumeration. Automation replaces human enumeration.
- This distinction matters: synchronous-derived integer-ratio clock pairs
  can use simpler synchronisers than fully async pairs — that relationship
  belongs in FAD §4.5; which specific flops use it does not.
- CDC gate enforcement: every Vivado-reported crossing must use a pattern
  on the approved-mechanism list. Unapproved pattern → waiver with
  quasi-static proof, or ADR extending the list.

### Execution-flow fixes

- `04-execution-flow.md` carried 5 stale CDC-philosophy mentions ("actual
  CDCs live in FAD §4.5 inventory") — all corrected to "architecture-level
  policy" wording.
- `MDS` → `MS` (11 occurrences); `rtl_ready` → `design_ready` (2 occurrences);
  `rtl_ready_blocking` → `design_ready_blocking` (1 occurrence);
  `§11.7 fully checked` → `§Test plan hooks / Design-ready self-check fully
  ticked` (1 occurrence). Section cross-refs (`MDS §3.3`, `§7.3`, `§12.1`,
  `§12.3`) converted to named-section references (section names are more
  stable than numbers in an unnumbered template).

### Waivers README

- Added one-liner definition at top: "A waiver is a version-controlled
  engineering judgement that overrides a specific tool finding, with stated
  reasoning, owner, and expiry trigger. Not a silencer."
- CDC review wording updated to match new philosophy: "every waived crossing
  must cite an approved-mechanism analysis or quasi-static proof per FAD §4.5
  policy" (was "FAD §4.5 cross-checked" — too vague under the new model).
- Forward reference to execution-flow doc sharpened: `#13-waiver-policy`
  anchor added.

### Top-level README

- `MDSs` → `MSs` in layout table.
- `waivers/` added to both layout tree and folder table (was in table only).
- New `## Methodology` section added: 5-row table linking all five
  `docs/methodology/` documents with one-line purpose each. Lead sentence
  "Start here before touching `arch/` or `rtl/`" establishes entry point
  for new contributors.

## Decisions

- **CDC mechanism removed from spec template.** Project-wide mechanism policy
  lives only in FAD §4.5. Per-spec "Domain crosses" column is sufficient.
  To record in ADR-0002 (clock architecture) when authored.
- **Two template variants maintained.** Annotated version for Gastón review
  handoff; clean version for repo commit as `arch/modules/_template.md`.
- **Named section references preferred over §N.N numbers** in cross-document
  citations within execution-flow and signoff-criteria, because the clean
  template has no numbered sections.
- **`04-execution-flow.md` bumped to v0.3.** All stale terminology resolved
  in this session.

## Open questions

- Gastón confirmation on template still pending (Task 8 — critical path).
  The annotated template is ready to share; his review unblocks ADR-0005
  acceptance and pilot module spec.
- `05-signoff-criteria.md` not yet swept for `MDS` / `rtl_ready` stale
  terminology — flagged at end of session, not yet done.
- `waivers/README.md` is policy-complete but not operationally complete:
  a concrete file-format example (one lint waiver, one CDC waiver, one
  timing waiver) is deferred until the lint tool is pinned (EXEC-FLOW-001/004).
- ADR-0002 (clock architecture) scope should explicitly state FAD §4.5 is
  architecture-level only; per-flop enumeration is `report_cdc` territory.

## Action items

- [ ] Share annotated `_template.md` with Gastón for review (Task 8)
- [ ] Commit `_template_clean.md` as `arch/modules/_template.md` (Task 10 final)
- [ ] Commit `04-execution-flow.md` v0.3
- [ ] Commit updated `waivers/README.md`
- [ ] Commit updated `README.md` (top-level)
- [ ] Sweep `05-signoff-criteria.md` for MDS / rtl_ready stale terminology
- [ ] When ADR-0002 is drafted: record CDC philosophy (FAD §4.5 = policy only; flop-pair enumeration = Vivado)
- [ ] Accept ADR-0005 once Gastón confirms template (Task 9)

## Artefacts produced

- `arch/modules/_template.md` — production spec template (clean, no scaffolding)
- `projects/bpms-sfu-architecture/planning/_template_annotated.md` — annotated version for Gastón review handoff
- `docs/methodology/04-execution-flow.md` — v0.3, CDC philosophy + terminology fixes
- `waivers/README.md` — updated with one-liner definition + CDC wording fix
- `README.md` (repo root) — methodology section added, MDS→MS, waivers/ in tree

## Re-seed block

GOAL: Complete Task 8 (Gastón template confirmation), accept ADR-0005, and begin FAD §2/§6 update to spec-level granularity.
CONSTRAINTS:
- Target device: BittWare RFX-8440A (AMD XCZU43DR-2FFVE1156E, speed grade -2)
- Source of truth: BPMS-1.0-SFU-001 v1.6 (primary), BPMS-1.0-ARCH-001 v2.4 (secondary)
- Spec template: `arch/modules/_template.md` — production version finalised this session
- CDC mechanism lives only in FAD §4.5 (policy), not per spec; Vivado does per-flop enumeration
- Terminology: MS (not MDS), design_ready (not rtl_ready), design_ready_blocking
PRIOR CONCLUSIONS:
- Template has 6 architect-side additions on top of Gastón's baseline; CDC mechanism sub-section removed
- Annotated template ready to share with Gastón; clean template ready to commit
- `04-execution-flow.md` v0.3 terminology-clean; `05-signoff-criteria.md` sweep still pending
- ADR-0005 in `proposed` state; flips to `accepted` on Gastón confirmation
CURRENT QUESTION: Get Gastón's sign-off on the spec template so ADR-0005 can be accepted and the pilot module spec can begin.
