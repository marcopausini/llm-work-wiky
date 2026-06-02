---
id: 2026-04-29--bpms-sfu-architecture--fpga-arch-reviewer-setup
date: 2026-04-29
project: bpms-sfu-architecture
type: decision
status: resolved
topics: [methodology, llm-workflow, chatgpt, review-process, adr]
source_chat: claude-opus-4.7
---

# fpga-arch-reviewer ChatGPT project setup

## Context

Final task of the *Tool and workflow setup* cluster (Task #18). The architect-side LLM
workflow defined by `02-architect-workflow.md` §6 calls for three roles: `fpga_arch`
(drafter, this Claude Project), `fpga_arch_reviewer` (adversarial reviewer, separate
ChatGPT project), `repo_operator` (Claude Code). The first two needed to be operational
before the first FAD review; this session stood up the second.

## Key findings

- Reviewer mandate (issue table only, no rewrites by default, no requirement-invention,
  no agreeable smoothing) is fixed by `02-architect-workflow.md` §6.4–§6.6 and was
  encoded verbatim into the ChatGPT custom instructions rather than reinterpreted.
- Source-of-truth hierarchy is the load-bearing anti-laundering measure. Seven tiers,
  binding (lower priority cannot override higher): SFU-001 → ARCH-001 → accepted ADRs →
  baselined FAD → frozen ICDs → design_ready MSs → AMD UG references.
- ChatGPT custom-instructions character budget makes detail in instructions a bad trade.
  Detail belongs in knowledge files; instructions are the operating manual that
  references them.
- Three-tier knowledge file split: mandatory (9 files, framework + sources, change
  rarely), active artefacts (rotate per session, replace not append), on-demand (UG949
  etc., upload only when a contested finding hinges on them).
- UG949/UG903/UG906 are explicitly on-demand, not mandatory. Citation by section is
  usually sufficient; loading the PDFs burns context against marginal benefit.
- Severity rubric is tied to gate consequence (blocker = gate cannot pass; high = gate
  passes only after closure; medium = pre-commit fix; low = deferrable). Maps cleanly
  to the §6.6 accept/reject/defer protocol.
- Reviewer must ask once when target gate or referenced document is missing; never
  guess. Single clarifying question is the only allowed exception to issue-table-only
  output.
- Tool configuration (LLM role definitions) is **not** committed to the design repo
  per `02-architect-workflow.md` §6.1; it lives in the architect's personal knowledge
  base.

## Decisions

- Default reviewer output: single Markdown issue table with columns Severity, Location,
  Issue, Action, Gate. Rewrites only on explicit request, and the issue table still
  comes first.
- Source hierarchy is binding, not advisory. Conflicts that cannot be resolved by
  hierarchy raise a `blocker` finding citing both sources.
- Source documents to be uploaded as `.docx` initially; conversion to `.md` deferred to
  REVIEWER-002 (decision after first review session, when citation-precision impact is
  measurable).
- Resolved review tables to be archived under `arch/reviews/<YYYY-MM-DD>-<artefact>.md`
  in the design repo (proposal, open as REVIEWER-003).
- Severity rubric encoded in instructions, not left to model judgement.

## Open questions

- REVIEWER-001 — Confirm ChatGPT custom-instructions character limit at project
  creation; trim §1 of the setup file if over budget.
- REVIEWER-002 — `.docx` vs `.md` for the two source documents in project knowledge.
- REVIEWER-003 — Confirm `arch/reviews/` archival convention.
- REVIEWER-004 — Whether UG949 PDF should be uploaded by default after one or two
  contested methodology findings appear.

## Artefacts produced

- `fpga-arch-reviewer-chatgpt-setup.md` — ChatGPT project instructions, knowledge file
  list (mandatory / active / on-demand), prompt template, three worked examples
  (FAD G1 / module spec G4a / ADR acceptance), operational notes, open items.
  Tool configuration; lives in personal knowledge base, not the design repo.

## Re-seed block

GOAL: instantiate the `fpga-arch-reviewer` ChatGPT project from the setup spec and run
its first review against FAD §1–§6 once that draft is updated.

CONSTRAINTS:
- Reviewer mandate fixed by `02-architect-workflow.md` §6.4–§6.6
- Tool configuration not committed to design repo (§6.1)
- Source hierarchy in setup spec §1 is binding; cannot be relaxed in a session
- Issue table is the only default output; rewrites on explicit request only

PRIOR CONCLUSIONS:
- Mandatory knowledge files are 9: SFU-001, ARCH-001, four methodology .md files,
  arch/README.md, arch/modules/_template.md, fpga-prj-best-methodology.md
- UG949/UG903/UG906 are on-demand, not mandatory
- Severity rubric ties to gate consequence (blocker/high/medium/low)
- Resolved review tables archive under `arch/reviews/<YYYY-MM-DD>-<artefact>.md`
- ADR-0005 (spec/uarch split) is the load-bearing prior decision driving the
  reviewer's design-ready checks

CURRENT QUESTION: which of REVIEWER-001..004 must be resolved before the first review
session, and which can be deferred to after first contact with the actual reviewer
behaviour?
