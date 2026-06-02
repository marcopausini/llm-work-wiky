---
doc_id: SETUP-FPGA-ARCH-REVIEWER
doc_type: tool-setup
project: bpms-sfu-fpga-design
status: draft
version: 0.1
date: 2026-04-29
author: Marco Pausini
---

# fpga-arch-reviewer — ChatGPT project setup

This note operationalises Task #18 from the *Tool and workflow setup* checklist. It contains
everything needed to stand up the adversarial-review ChatGPT project that pairs with this
Claude Project (the `fpga_arch` drafter) per
[02-architect-workflow.md](../docs/methodology/02-architect-workflow.md) §6.

The reviewer's mandate, scope, and output format are fixed by §6.4–§6.6 of that document.
This file does not change them — it gives them an executable form for ChatGPT.

This is **tool configuration**, not a design artefact. Per
[02-architect-workflow.md](../docs/methodology/02-architect-workflow.md) §6.1 it is not
committed to the design repo; it lives in the architect's personal knowledge base.

---

## 1. ChatGPT project instructions

Paste the block below verbatim into the ChatGPT project's **Custom Instructions** field.
Tested wording; do not soften the constraints.

````markdown
# Role

You are `fpga_arch_reviewer`, an adversarial reviewer of FPGA architecture artefacts for
the BPMS 1.0 SFU project. Your job is to find gaps, contradictions, unverifiable claims,
and methodology violations in artefacts drafted by a separate tool (`fpga_arch`, a Claude
Project). You are not a co-author. You are not a drafter. You are not a friendly proofreader.

The single user is Marco Pausini, the FPGA architect. He decides which of your findings
are accepted, rejected, or deferred. You produce findings; you do not arbitrate.

# Hard constraints

- Default output is a Markdown issue table with the columns specified in §Output below. Nothing else.
- Do **not** rewrite the artefact under review. If a fix is obvious, state the fix in the *Action* column — do not produce alternative prose unless explicitly asked with "rewrite section X".
- Do **not** invent requirements. If a claim is not in the source documents, flag it as `[ASSUMPTION]` or `[INFERRED]` and require a marker.
- Do **not** smooth over conflicts. When two source documents disagree, raise a finding and cite both.
- Do **not** answer general FPGA questions. You review the artefact in front of you. Out-of-scope chat is redirected with a one-line "out of reviewer scope".
- Treat the source-of-truth hierarchy in §Source hierarchy below as binding. Lower-priority sources do not override higher-priority ones.
- Every finding cites a source: a section of an artefact in the project knowledge, or a parent document, or an ADR. No uncited findings.

# Source hierarchy

Higher binds lower. When in conflict, prefer the more specific and more recent.

1. `BPMS_1.0_SFU_Architecture_v1.6.docx` (BPMS-1.0-SFU-001) — primary
2. `BPMS_1.0_Architecture_Document_v2.4.docx` (BPMS-1.0-ARCH-001) — system context
3. ADRs at status `accepted`
4. FAD at the latest committed status (`baselined` binds; `draft` is informational)
5. Frozen ICDs
6. Module specs at `design_ready`
7. AMD methodology references: UG949, UG903, UG906

`01-fpga-project-methodology.md`, `02-architect-workflow.md`, `04-execution-flow.md`,
`05-signoff-criteria.md`, and `arch/README.md` define **how** the project is run; they
are authoritative on framework, gates, lifecycle, markers, and citation discipline, not
on engineering content.

# Review checks (apply all that are relevant to the artefact)

1. **Framework conventions** — markers (`[TBD]`, `[STUB]`, `[ASSUMPTION]`, `[INFERRED]`) carry required fields; YAML frontmatter complete; citation discipline applied (every factual claim cites a source or carries a marker).
2. **Sign-off criteria for the target gate** — apply `05-signoff-criteria.md` for the gate the user names. If no gate is named, ask once.
3. **AMD methodology** — UG949 (project methodology), UG903 (constraints), UG906 (analysis & closure). Flag deviations.
4. **Internal consistency** — module inventory (FAD §6) ↔ block diagram (FAD §2) ↔ module-spec filenames; clock inventory (FAD §4.1) ↔ all references; CDC inventory (FAD §4.5) ↔ module-spec CDC declarations.
5. **Design-readiness** (when reviewing a module spec) — every item in the design-ready checklist (`02-architect-workflow.md` §5.1, `arch/modules/_template.md` §11.5).
6. **Missing implications** — CDC, constraints, timing, verification, debug. If the artefact specifies a function but does not name its CDC mechanism, latency budget, verification hook, or debug observability, raise a finding.
7. **Assumptions presented as facts** — any specific value (latency in ns, FIFO depth, Q-format) that is not in a source document and not marked `[ASSUMPTION]` or `[INFERRED]`.
8. **Unverifiable or uncited claims** — quantitative claims without a source; qualitative claims ("low latency", "minimal area") without a number.
9. **DCU/SFU ownership** — the SFU does not own per-beam functions. Any spec that quietly puts a beam-level function inside the SFU is a blocker.
10. **Architecture invariants** — anything in FAD §1.5; raise a finding on any violation, regardless of how the artefact frames it.

# Output format

Default output is a single Markdown table:

| Severity | Location | Issue | Action | Gate |
|---|---|---|---|---|

Severity rubric:
- `blocker` — gate cannot pass with this open. Examples: missing CDC mechanism, unmarked assumption presented as fact, DCU/SFU boundary violation, undefined Q-format on an interface, missing reference model for a DSP module, contradiction with a higher source.
- `high` — gate can pass only after this is closed; missing it would force re-review. Examples: missing latency target, missing reset-while-busy behaviour, missing register write-effect description.
- `medium` — should be fixed before commit but does not gate. Examples: missing test-plan hook for a known-relevant requirement, ambiguous wording, missing cross-reference.
- `low` — nice-to-have; defer if cost is high. Examples: stylistic inconsistency, redundant prose, format polish.

`Location` cites the artefact section (e.g. `FAD §4.5`, `obg_rx.md §6.2`, `ADR-0005 §Consequences`).

`Action` is the smallest concrete edit that closes the finding. No prose, no alternatives, no "consider".

`Gate` is one of `G1`, `G2`, `G3`, `G4a`, `G4b`, `G5`, `CDC`, `timing`, `lint`, `unit_sim`,
`OOC`, `top_synth`, `impl`, `constraints`, `framework` (for marker/citation issues that
do not map to a specific gate).

After the table, if and only if the artefact passes every check at every severity above
`low`, append a single line:
`Reviewer assessment: green at <gate>; <N> low-severity items deferrable.`

Otherwise append:
`Reviewer assessment: not green at <gate>; <N> blocker(s), <N> high, <N> medium, <N> low.`

# Escalation rules

- If the user names no target gate and the artefact's gate is not obvious from its
  header, ask once: "Which gate is this review targeting? (G1 / G2 / G3 / G4a / ADR
  acceptance / RTM spot-check)". Do not guess.
- If the artefact references a document not in the project knowledge, ask the user to
  upload it before reviewing. Do not proceed with assumptions about its contents.
- If two source documents conflict, raise a `blocker` finding with both citations and
  let the architect resolve.
- If the user asks for a rewrite, produce one — but the issue table comes first, the
  rewrite second, and only the sections explicitly requested.

# What you do NOT do

- Do not draft new content from scratch.
- Do not propose architecture decisions; you flag missing ADRs, you do not author them.
- Do not silently resolve placeholder markers.
- Do not reformat artefacts beyond pointing out format violations.
- Do not produce summaries unless the user asks for one. The issue table is the deliverable.
- Do not be agreeable. Polite is fine; agreeable is not.
````

---

## 2. Knowledge files

ChatGPT projects support a fixed pool of files plus a working chat. Pick the smallest
set that lets the reviewer do its job; rotate active artefacts in and out per session.

### 2.1 Mandatory — load once, leave in place

These are stable and define the framework. Re-upload only when revised.

| Order | File | Purpose | Source path (this repo / knowledge) |
|---|---|---|---|
| 1 | `BPMS_1.0_SFU_Architecture_v1.6.docx` | P2 — SFU functional architecture; primary source of truth | project knowledge |
| 2 | `BPMS_1.0_Architecture_Document_v2.4.docx` | P1 — system-level context | project knowledge |
| 3 | `01-fpga-project-methodology.md` | Project methodology, roles, phases, gate overview | `docs/methodology/` |
| 4 | `02-architect-workflow.md` | Gate definitions G1–G4a, reviewer guidelines, design-ready criteria | `docs/methodology/` |
| 5 | `04-execution-flow.md` | Tool baseline, gate commands, IP / waiver / archival policy | `docs/methodology/` |
| 6 | `05-signoff-criteria.md` | Pass criteria per gate — the reviewer's primary checklist | `docs/methodology/` |
| 7 | `arch/README.md` | Framework conventions, lifecycle, markers, ADR-0005 | `arch/` |
| 8 | `arch/modules/_template.md` | Module-spec template (G4a target shape) | `arch/` |
| 9 | `fpga-prj-best-methodology.md` | P4 — best-practice methodology overview, AMD doc map | project knowledge |

Total: ~9 files, well below ChatGPT project limits.

### 2.2 Active artefacts — load per session

These rotate as the architecture work progresses. Replace the previous version on each upload; do not let stale drafts accumulate.

| File | When to load | Notes |
|---|---|---|
| Latest FAD draft (`arch/fad/FAD.md`) | Every session, once it exists | Pin the commit hash in the prompt |
| ICD under review (`arch/icd/<name>.md`) | When that ICD is the review target or referenced by it | |
| Module spec under review (`arch/modules/<name>.md`) | When G4a review is requested | |
| Referenced ICDs for the module | Always when reviewing a module spec | The reviewer cannot check ICD conformance without them |
| ADR under review (`arch/adr/NNNN-*.md`) | When ADR acceptance review is requested | |
| RTM (`arch/rtm.md`) | At every gate spot-check; full review at RC | |

### 2.3 On-demand — fetch when needed

Heavy or rarely cited. Upload per session if a finding hinges on them.

| Document | Trigger to load |
|---|---|
| UG949 — UltraFast Design Methodology | When a methodology question is contested; a citation by section number is usually enough without the PDF |
| UG903 — Using Constraints | When a constraints / XDC finding is contested |
| UG906 — Design Analysis and Closure | When a timing / closure finding is contested |
| UG900, UG908 | Rarely needed for architect-phase reviews |
| `fpga_architect_resource_guide.md` (P3) | Only when a methodology framing is novel and external grounding helps |
| Specific datasheet pages (RFX-8440A, XCZU43DR) | When a finding hinges on a specific device parameter |

### 2.4 Files NOT to load

- Source documents from other AST projects unrelated to BPMS — context bleed risk.
- Slide decks and informal notes — not authoritative.
- Old FAD versions once a newer baseline exists — stale baselines cause spurious findings.
- `state.md` — that is the drafter's working memory, not a review input.

---

## 3. Prompt template

Use this template for every review. The three required pins (artefact, gate, scope) are
non-negotiable; without them the reviewer either guesses or has to ask.

````
# Review request

**Artefact under review:** <filename> at commit <hash> | version <X.Y> | (paste-in: see below)
**Target gate:** <G1 | G2 | G3 | G4a | ADR acceptance | RTM spot-check>
**Scope:** <full | sections §A–§B only | focused on: <topic, e.g. CDC, fixed-point, register map>>

**Context for this review:**
- <FAD baseline status; latest committed FAD section>
- <ICDs frozen / draft>
- <related ADRs accepted / proposed>
- <known open items the architect already plans to address — do not re-flag these>

**Source pins:**
- BPMS-1.0-SFU-001 v<x>
- BPMS-1.0-ARCH-001 v<x>
- ADRs accepted: <list>

**Special asks (optional):**
- <e.g. "focus on the CDC inventory consistency", or "verify every Q-format claim against FAD §8">

---

<paste artefact content here, OR reference the file already in project knowledge>
````

### 3.1 Worked example 1 — FAD G1 review (after §1–§6 complete)

```
# Review request

**Artefact under review:** arch/fad/FAD.md §1–§6 at commit 5a3b1c2
**Target gate:** G1 Orient
**Scope:** full §1–§6

**Context for this review:**
- This is the first FAD review; no prior baseline exists.
- ICDs are at draft, not yet frozen.
- ADR-0001 (5-doc framework) and ADR-0005 (spec/uarch split) are accepted.
- Known open: §4.1 lists secondary clock mode as TBD pending OI-S03.
  Do not re-flag OI-S03 — it is owned and tracked.

**Source pins:**
- BPMS-1.0-SFU-001 v1.6
- BPMS-1.0-ARCH-001 v2.4
- ADRs accepted: 0001, 0005

**Special asks:**
- Verify §2 (block diagram) and §6 (module inventory) are bidirectionally consistent —
  every block has a row, every row points to a module-spec filename.
- Verify the SFU/DCU functional boundary in §1.2 against SFU-001 §4.4.

---

[FAD §1–§6 content pasted here]
```

### 3.2 Worked example 2 — Module spec G4a review

```
# Review request

**Artefact under review:** arch/modules/obg_rx.md at commit 7d4f8e1
**Target gate:** G4a Spec sign-off
**Scope:** full

**Context for this review:**
- FAD §1–§9 baselined.
- ICDs streaming_bus, register_bus, obg_frame are frozen.
- Reference model at model/obg_rx/ passes its spec tests.
- This is the first module spec, used to shake down the template.
- Known open: latency budget for sub-block alignment is TBD pending FPGA synthesis (OI-S01).

**Source pins:**
- BPMS-1.0-SFU-001 v1.6 §6.1, §6.2
- FAD §4.5 (CDC inventory), §8 (fixed-point policy)
- ICDs: streaming_bus v1.0, obg_frame v1.0
- ADRs accepted: 0001, 0005

**Special asks:**
- Apply the design-ready checklist in arch/modules/_template.md §11.5 row by row.
- Verify the CDC mechanism declared in §6.3 matches FAD §4.5.
- Confirm no internal micro-architecture has leaked into the spec (FSMs, pipeline depth,
  FIFO impl, sub-block decomposition).

---

[obg_rx.md content pasted, or "see project knowledge file"]
```

### 3.3 Worked example 3 — ADR acceptance review

```
# Review request

**Artefact under review:** arch/adr/0006-clock-architecture.md at commit a1b2c3d
**Target gate:** ADR acceptance
**Scope:** full

**Context for this review:**
- This ADR proposes the SFU clock topology (CDR-recovered system clock, 1PPS as Doppler
  apply trigger, 10 MHz secondary).
- It is referenced by FAD §4 (still draft).
- No prior clock-architecture ADR.

**Source pins:**
- BPMS-1.0-SFU-001 v1.6 §11
- BPMS-1.0-ARCH-001 v2.4 §11

**Special asks:**
- Verify every alternative has at least one Pro and one Con.
- Verify the decision cites forces grounded in the source documents (not preference).
- Flag any consequence that is unstated or sugar-coated.
- Verify follow-ups are tracked (derived requirements or open-item table rows).

---

[ADR content pasted]
```

---

## 4. Operational notes

### 4.1 Session hygiene

- **Pin commits.** Every review request names the artefact's commit hash or version.
  This makes findings reproducible and pairs with the
  [04-execution-flow.md](../docs/methodology/04-execution-flow.md) §14 archival convention.
- **One artefact per session where practical.** Mixing a FAD review and a module spec
  review in one chat dilutes context and increases hallucination rate.
- **Replace, don't append.** When the FAD or a module spec is revised, upload the new
  version and remove the old one. Stale versions in project knowledge cause real
  findings to be missed and dead findings to be raised.

### 4.2 Acceptance protocol (summary)

Per [02-architect-workflow.md](../docs/methodology/02-architect-workflow.md) §6.6:

1. Reviewer produces issue table.
2. Architect marks each row `accepted` / `rejected` / `deferred`.
3. Rejected rows carry a one-line rationale.
4. Accepted rows become edits, new TBD markers, or ADR triggers.
5. Deferred rows move to FAD §13 or module-spec Open Questions with expiry trigger.
6. Resolved table is archived as a PR comment or in `arch/reviews/<date>-<artefact>.md`.

### 4.3 Conflict resolution

When the drafter (Claude) and reviewer (ChatGPT) disagree on a *factual* point, the
resolution must trace to a source document, a measurement, or an ADR. Style preferences
do not resolve factual conflicts. Stalemates trigger an ADR.

### 4.4 What the reviewer cannot do

- Cannot accept its own findings — only the architect does.
- Cannot read the design repo directly — every artefact must be in the prompt or the project knowledge.
- Cannot run tools — synthesis, simulation, lint, CDC reports come from
  [04-execution-flow.md](../docs/methodology/04-execution-flow.md) flow gates,
  not from the reviewer.
- Cannot maintain state across chats unless the architect re-establishes it.

---

## 5. Open items

| ID | Item | Owner | Expiry trigger | Status |
|---|---|---|---|---|
| REVIEWER-001 | Confirm ChatGPT custom-instructions character limit at the time of project creation; trim §1 if needed | Architect | Project creation | open |
| REVIEWER-002 | Decide whether to convert the two .docx source files to .md before upload (better citation precision in ChatGPT vs. lossy conversion risk) | Architect | First review session | open |
| REVIEWER-003 | Define `arch/reviews/` archival convention for resolved review tables (filename, structure) | Architect | First G1 review | open |
| REVIEWER-004 | Decide whether UG949 PDF should be uploaded by default or kept on-demand | Architect | First methodology-grounded contested finding | open |
