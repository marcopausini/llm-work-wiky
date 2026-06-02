---
id: methodology-presentation-outline
date: 2026-04-27
project: bpms-sfu-architecture
type: presentation-outline
status: draft
version: 0.2
audience: [fpga-architect, rtl-designer, dv-engineer]
target_format: pptx
target_duration_min: 30
estimated_slide_count: 20
coverage_balance: methodology_architect_80_exec_signoff_20
author: Marco Pausini
related:
  - 2026-04-26--bpms-sfu-architecture--project-methodology-and-spec-uarch-split.md
  - docs/methodology/01-fpga-project-methodology.md
  - docs/methodology/02-architect-workflow.md
  - docs/methodology/04-execution-flow.md
  - docs/methodology/05-signoff-criteria.md
---

# BPMS 1.0 SFU FPGA — Project Methodology

## Narrative outline for next-meeting presentation (v0.2)

---

## Meta

**Meeting goal.** Get internal alignment on (i) the project methodology, (ii) the
roles each engineer owns, (iii) the architect-workflow detail that drives the next
6 weeks, (iv) the execution + sign-off framework, (v) the schedule. Surface and
resolve the open decisions listed in the Appendix.

**Audience and what each takes away.**

| Attendee | Walks away with |
|---|---|
| System architect (Shlomi) | Confidence in approach + 6-week schedule with stated contingencies; clear list of his asks (OI ownership). |
| RTL designer (Gastón) | Exact contract he receives at G4a, exact contract he owns at G4b; agreement (or red-flag) on the spec template. |
| DV engineer | What they consume at G5, when each artefact stabilises, what the verification contract looks like. |
| Architect (presenter) | ADR-0005 momentum toward `accepted`; agreement on review cadence and reviewer LLM tool. |

**Time budget.** ~30 min talk + 15 min discussion. Coverage: ~80% methodology +
architect workflow, ~20% execution flow + sign-off.

**Slide count.** 20 slides across six sections.

**Build philosophy.** Compressed from v0.1 (28 slides). Architect-workflow detail is
preserved by merging closely-related slides; methodology framing tightened to two
slides; new section covers exec flow + sign-off in three slides.

---

## Section A — Frame (3 min · 2 slides)

### Slide A.1 — Title and meeting goal

**Key message.** This meeting is the methodology agreement point — not a status
update.

**On slide.**

- Title: *BPMS 1.0 SFU FPGA — Project Methodology*
- Subtitle: *Roles, gates, and the architect workflow*
- Date and author
- Three audience bullets (one per role) directly under the subtitle:
  - *Architect / sponsor* — agree on schedule and methodology
  - *RTL designer* — agree on the spec template + handoff contract
  - *DV* — agree on what gets consumed at G5

**Speaker notes.** Open with what's at stake: by end of this meeting we want to leave
with M1 committed (or pushed back with reasons), ADR-0005 either accepted or with
the exact blocker named, and the spec template confirmed. Set expectation that the
bulk of the talk is the architect workflow because that's what we're committing to
deliver against.

**Time.** ~1 min.

### Slide A.2 — Why now + what's new

**Key message.** The framework was specification-only. We now have an *execution*
layer to make the schedule real and to align with Gastón's existing flow.

**On slide.**

- We started with `arch/` — a 5-doc Markdown framework. Good for *what to write*,
  silent on *how to build it*.
- Need to align with the RTL designer's existing LLM-assisted RTL flow (Claude Code).
- System architect asked for a delivery date — schedule needs grounding in a
  methodology, not a folder layout.

**What to show.** Small "before / after" diagram:

- *Before*: 5 boxes (FAD / MS / ICD / ADR / RTM)
- *After*: same 5 boxes inside a "spec layer" lane, with a second "execution layer"
  lane below containing the 5 methodology docs (`01–05-*.md`)

**Speaker notes.** Keep this short. The point is just to motivate why we're not
just talking templates.

**Time.** ~2 min.

---

## Section B — Methodology framework (4 min · 2 slides)

### Slide B.1 — Two-layer framework + phases

**Key message.** Spec layer says *what* the architecture is; execution layer says
*how it gets built and what passing means*. Five phases, each producing artefacts
the next phase consumes.

**On slide — split into two compact panels.**

*Panel 1 — Two layers:*

| Layer | Location | Key docs | Owner |
|---|---|---|---|
| Spec | `arch/` | FAD, MS, ICD, ADR, RTM (+ `arch/README.md`) | Architect |
| Execution | `docs/methodology/` | `01–05-*.md` | Shared (architect-led) |

*Panel 2 — Phases:*

| # | Phase | Key gate |
|---|---|---|
| 1 | Architecture | G1, G2, G3 |
| 2 | Module specification | G4a |
| 3 | RTL implementation | G4b + integration |
| 4 | Integration | impl sign-off |
| 5 | HW bring-up | bring-up sign-off |

**Speaker notes.** This folder defines *what* the architecture is; the methodology
docs define *how* it gets built. Resist requests to merge them. The split is what
allows specification to baseline before tooling is finalised. Phases overlap —
gates, not calendars, control promotion.

**Time.** ~2 min.

### Slide B.2 — Repo layout (one slide)

**Key message.** Flat by artefact type. Reference models live in the *design* repo
because they are part of the spec.

**On slide.** Compact directory tree of `bpms-sfu-fpga-design/` (top level only) +
one-liner for the verif repo. Highlight `model/` and `tb/` as architect-vs-designer
boundary points. Highlight `docs/methodology/` as the new execution-layer location.

**Speaker notes.** Three points only:

1. Flat by artefact type, not by lifecycle phase.
2. Design and verification are *separate* repos. The verif repo consumes `arch/`
   read-only.
3. Reference models live with specs, not with verification. They're how the
   architect makes "bit-exact" enforceable.

**Time.** ~2 min.

---

## Section C — Roles and the spec/uArch split (5 min · 3 slides)

### Slide C.1 — Three roles, three sets of artefacts

**Key message.** Each role owns *what work products it produces*, not org-chart
positions. A single engineer may fill multiple roles.

**On slide — table.**

| Role | Owns | Gates owned |
|---|---|---|
| FPGA Architect | FAD, MS, ICD, ADR, RTM, refmodels (DSP) | G1, G2, G3, G4a |
| RTL Designer | uArch, RTL source, unit TBs, impl evidence | G4b, integration |
| Verification Engineer | UVM env, integration TB, coverage closure | G5 |

**Speaker notes.** Note explicitly: the architect does *not* own the uArch
document. That's a deliberate role boundary, codified in ADR-0005. The three roles
are about work products, not job titles — Marco may fill architect + DV in week 1,
Gastón may fill designer + integration in week 5.

**Time.** ~1.5 min.

### Slide C.2 — The spec/uArch split (ADR-0005)

**Key message.** The MS is *what* the block does externally. The uArch is *how* it
does it internally. Different documents, different owners, different gates.

**On slide — side-by-side table.**

| | Module Spec (MS) | Micro-Architecture (uArch) |
|---|---|---|
| Filename | `arch/modules/<m>.md` | designer's choice (e.g. `rtl/<m>/uarch.md`) |
| Owner | Architect | RTL Designer |
| Scope | External contract | Internal implementation |
| Lifecycle gate | G4a | G4b |
| Granularity | Functional block (~10 for SFU) | One or more `.sv` files inside the block |
| Examples | ports, interfaces, clocks, registers, operation, performance, errors, test hooks, refmodel pointer | FSMs, pipeline depth, FIFO depths, sub-block decomposition, internal Q-formats |

**Speaker notes.** Spend 90 seconds here — this is the slide everyone needs to
remember. Reference ADR-0005 by name. Note that the SFU's ~10 functional blocks
each may decompose into multiple `.sv` files; the architect doesn't dictate that
decomposition.

**Time.** ~2 min.

### Slide C.3 — Reference model ownership + why the split matters

**Key message.** For DSP/numerical blocks, the architect authors a Python bit-exact
reference model. The refmodel *defines the acceptance criterion* — it's part of the
spec, not a verification artefact. The split unblocks parallelism.

**On slide — two compact panels.**

*Refmodel ownership:*

- Refmodels live in `model/<m>/` in the *design* repo.
- Bit-exactness policy stated in MS §3.3: `bit_exact` / `ulp_bounded(N)` /
  `not_applicable`.
- Designer's unit TB correlates RTL against refmodel sample-by-sample (for
  `bit_exact`).
- Vendor IP wrappers are `not_applicable` — boundary metric used instead.

*Why the split matters:*

- Architect can baseline contracts before designer commits to an implementation →
  DV can build TBs in parallel with RTL.
- Designer keeps freedom on internal choices → faster iteration, better
  implementations.
- Reduces architect/designer churn: external questions belong in MS; internal
  questions don't reach the architect.

**Speaker notes.** This is the answer to "how do we make 'bit-exact' enforceable?"
The architect signs the refmodel as authoritative; the designer can't argue with
golden vectors. Failure-mode test: if the designer asks "what should the FSM look
like?" → uArch question. If they ask "what does this block output when input is X?"
→ MS question, and if the answer isn't there, the spec isn't `design_ready`.

**Time.** ~1.5 min.

---

## Section D — Architect workflow deep dive (12 min · 8 slides)

This is the core of the talk. Architect responsibilities, document authoring,
gates G1–G4a, the LLM-assisted pipeline, the failure modes. Compressed from 15
slides in v0.1 to 8 by merging closely-related content.

### Slide D.1 — Architect workflow at a glance + phase flow

**Key message.** Inputs → outputs → gates owned. Work flows in dependency order.

**On slide — two compact panels.**

*Inputs / Outputs / Gates:*

| Inputs | Outputs | Gates |
|---|---|---|
| ARCH-001 v2.4 (secondary) | 1 FAD | G1 Orient |
| SFU-001 v1.6 (primary) | ~10 MSs | G2 Contracts |
| Vendor docs (RFX-8440A, XCZU43DR, FB spec) | ~5 ICDs | G3 Budgets |
|  | ADRs as needed | G4a Spec sign-off |
|  | 1 living RTM | + consistency review at G4b |
|  | Refmodels for DSP blocks |  |

*Phase flow (small block diagram):*

```
FAD §1–§6 (Orient)
   → Core ICDs (streaming_bus, register_bus, obg_frame)
      → FAD §7–§12 (Contracts + Budgets + remaining)
         → Module Specs (one per functional block from §6 inventory)
            → RTM (seeded day 1, updated continuously)
```

**Speaker notes.** Set the scope before mechanics. ICDs come *between* the orient
sections and the contracts/budgets sections — module specs need frozen ICDs, and
contracts sections reference them.

**Time.** ~2 min.

### Slide D.2 — Section-by-section drafting + markers + citation

**Key message.** Sections, not whole documents. Each section is a unit of review,
revision, and commit. Visible, owned gaps beat hidden gaps.

**On slide — three compact panels.**

*Drafting cadence (10 steps, two columns):*

Before drafting:

1. Identify target template section
2. Gather source material
3. List specified vs inferred vs proposed

During drafting:

4. Fill template; cite sources inline
5. Mark every gap
6. Identify ADR triggers
7. Identify cross-references

After drafting:

8. Self-check against gate criteria
9. Submit for independent review
10. Revise; commit

*Placeholder markers (compact):*

- `[TBD: <reason>, <owner>]`
- `[STUB: <blocking item>]`
- `[ASSUMPTION: <text>, <expiry trigger>]`
- `[INFERRED from <source §>]`

*Citation rule:* Every factual claim from a parent doc carries an inline citation:
`(SFU-001 §6.4)`. Otherwise mark as `[INFERRED]` or `[ASSUMPTION]`.

**Speaker notes.** The cadence sets the operating tempo: section-sized work units,
each fully closed before moving on. Resist whole-document drafts — they're not
reviewable. The markers are *greppable*. They enforce the specified / inferred /
proposed split. Citations are the anti-laundering principle: with LLMs in the
pipeline, the failure mode is plausible-but-wrong text.

**Time.** ~1.5 min.

### Slide D.3 — Architecture invariants and the functional boundary

**Key message.** Some constraints apply across the entire design. The DCU/SFU
ownership boundary is one of them. The architect enforces; ADRs record exceptions.

**On slide.**

- FAD §1.5 lists invariants. No section, MS, ICD, or ADR may violate them.
- DCU/SFU functional boundary is load-bearing: the SFU does *not* do per-beam work.
- Boundary-crossing procedure:
  1. Open an ADR.
  2. Cite the system-level source that justifies the change.
  3. If no source exists → reject by default.

**Speaker notes.** Concrete example: anything that looks like per-beam Doppler in
the SFU is an immediate red flag. The DCU owns it. If a proposal moves it, that's
an ADR conversation with explicit system-level justification.

**Time.** ~1 min.

### Slide D.4 — Design-ready criteria (the G4a checklist)

**Key message.** A module spec is `design_ready` only when the external contract
is complete and unambiguous. The completeness test is concrete.

**On slide.**

*Checklist:* clock domains · interfaces · parameters · register set · operation ·
performance · error/status · system constraints · test plan hooks · refmodel.

*Completeness test (large, central on slide):*

> Could a senior RTL designer implement this module from the spec + referenced ICDs,
> making their own micro-architecture choices, without asking the architect a
> question about intended *external* behaviour?
>
> If no — `not_design_ready`.

**Speaker notes.** Read the test out loud. This is the line we're going to use
hundreds of times over the next 6 weeks. It's also the line that protects against
scope creep into uArch. **Pre-align with Gastón before the meeting** — if he
disagrees with this test, the whole methodology shifts.

**Time.** ~2 min.

### Slide D.5 — What the MS does NOT prescribe + common failure modes

**Key message.** The other half of the spec/uArch split. Specs fail in predictable
ways.

**On slide — two compact panels.**

*Not in the MS (uArch territory):*

- FSMs / state machine internals
- Pipeline depth or staging
- FIFO implementation details
- Internal sub-block decomposition (number of `.sv` files)
- Internal fixed-point precision (only boundary formats specified)
- Register address/bit assignment (architect specifies *minimum register set*)

*Common failure modes (table):*

| Failure | Fix |
|---|---|
| Ambiguous backpressure ("handles backpressure") | Specify: stall / drop / error |
| Performance target without a number | State budget in cycles or ns |
| Error condition listed but external behaviour undefined | Specify status / interrupt / output behaviour |
| Spec prescribes FSM states or pipeline depth | Remove — state external constraint, not implementation |

**Speaker notes.** Last failure mode is the most common for architects coming from
an RTL background. We'll catch each other on it. Only *boundary* fixed-point
formats are in the MS — internal precision (e.g. accumulator width inside a FIR)
is the designer's choice unless it observably violates a refmodel-defined
acceptance criterion.

**Time.** ~1.5 min.

### Slide D.6 — LLM-assisted workflow: roles + pipeline

**Key message.** Three LLM roles, one human authority. Five-step pipeline with the
human at "Decide".

**On slide — two compact panels.**

*Roles:*

| Role | Tool | Purpose |
|---|---|---|
| `fpga_arch` (drafter) | Claude.ai | Draft sections from sources |
| `fpga_arch_reviewer` | ChatGPT (separate project) | Adversarial review |
| `repo_operator` | Claude Code | Repo-aware edits + consistency checks |

Sign-off authority: human only.

*Pipeline:*

```
A. Draft (Claude.ai)        section + citations + markers
   ↓
B. Review (ChatGPT)         issue table by severity
   ↓
C. Decide (engineer)        accept / reject / defer
   ↓
D. Apply (Claude Code)      minimal diffs
   ↓
E. Check (Claude Code)      cross-refs, frontmatter, RTM
```

**Speaker notes.** Even when an LLM-generated artefact looks polished and complete,
*no LLM signs off*. Steps B and E both use LLMs but with no rewrite authority.
Step D is the only LLM step that writes to the repo, and only when given accepted
review items.

**Time.** ~2 min.

### Slide D.7 — Live demo: real reviewer issue table

**Key message.** Reviews come back as a structured issue table, not as rewritten
prose. Easy to act on, hard to miss items.

**On slide — example table from an actual review (3-row excerpt).**

| Severity | Location | Issue | Action | Gate |
|---|---|---|---|---|
| blocker | FAD §4.5 | CDC Aurora→DSP unspecified | Add async FIFO row, depth, owner | G1 / CDC |
| high | spec §Performance | Band gain Q-format undefined | Specify Q(a.b) and saturation rule | G4a |
| medium | spec §Operation | Reset-while-busy missing | Add drop/complete/error spec | G4a |

**Speaker notes.** This is the demo slide — pull a real 3-row excerpt from a review
already run on FAD §2 draft. Walk through one row showing how it became a commit.
If no real review exists yet by deck-build time, run one against the current FAD
§2 draft this week (action item).

**Time.** ~1.5 min.

### Slide D.8 — Architect at G4b + later phases

**Key message.** G4a is the architect's gate; G4b is the designer's. Architect
reviews G4b only for *architectural consistency*. Architect doesn't disappear after
Phase 2 — ~30–40% of architect time stays engaged through implementation and
bring-up.

**On slide — two compact panels.**

*Architect at G4b reviews for:*

- Implementation respects the MS contract
- CDC mechanisms match MS and FAD §4.5
- Resource usage within FAD §9.2 budget
- No undocumented functional boundary crossing

*Architect's later-phase responsibilities:*

- Phase 3 (RTL): cross-module integration questions, post-synth budget updates,
  ADRs for impl-driven changes
- Phase 4 (integration): top-level constraints, CDC reports, timing-closure
  adjudication
- Phase 5 (bring-up): bring-up sequence, hardware validation evidence

**Speaker notes.** Designer doesn't reach into the spec; architect doesn't reach
into the implementation. Exception: when implementation evidence reveals a missing
constraint — that goes back into the MS or an ADR. Set expectation now: architect
bandwidth doesn't go to zero after Phase 2.

**Time.** ~1.5 min.

---

## Section E — Execution flow + sign-off (4 min · 3 slides)

This is the new section per audience request. ~20% of the talk. Goal is overview
only — not deep dive — anchored in `04-execution-flow.md` and
`05-signoff-criteria.md`.

### Slide E.1 — Execution flow: build entrypoints and gate commands

**Key message.** Single `make`-based flow wraps Vivado, simulator, lint. Every flow
stage has a target, a report, and a sign-off section it points to.

**On slide — table (subset of the most-used targets from `04-execution-flow.md` §3).**

| Target | Stage | Sign-off ref |
|---|---|---|
| `make doc_check` | Documentation gate | §3 of signoff doc |
| `make lint MOD=<m>` | RTL lint | §4 |
| `make sim_unit MOD=<m>` | Unit sim (incl. refmodel correlation) | §5 |
| `make synth_ooc MOD=<m>` | OOC synthesis | §6 |
| `make cdc MOD=<m>` | CDC check | §7 |
| `make synth_top` / `make impl` / `make timing` | Top synth + implementation | §8, §9, §10 |

Reports archive under `reports/<commit-hash>/` — each gate references the commit
at which evidence was captured.

**Speaker notes.** This is the *how to run the build* layer. Two key principles:
(i) every stage produces an archived report at a commit hash, (ii) the gate command
in `04-execution-flow.md` always cross-references the matching pass criteria in
`05-signoff-criteria.md`. The flow is single-source — no manual Vivado clicks.

**Time.** ~1.5 min.

### Slide E.2 — Sign-off criteria: gate map

**Key message.** Every artefact has a defined "passing" definition. Sign-off is a
human responsibility — LLMs don't sign anything.

**On slide — gate map (compact version of `05-signoff-criteria.md` §2).**

```
                G1   G2   G3                       G4a               G4b   integ.    G5         HW
Architecture ──●────●────●──────────────────────────●─────────────────────────────────────────────
Spec / refmodel                                     ●(per module)
RTL lint                                                              ●(per module)
Unit sim                                                              ●(per module)
OOC synth                                                             ●(per module)
CDC                                                                   ●(per module) ●(top)
Constraints                                                                          ●(top)
Top synth                                                                            ●(top)
Implementation                                                                       ●(top)
DV / UVM                                                                                       ●
HW bring-up                                                                                              ●
```

**Speaker notes.** Walk through one column to show the structure (e.g., G4a =
spec sign-off per module = MS reaches `design_ready`). No gate may be waived
silently — waivers are recorded as ADRs (architecture-level) or `waivers/` entries
(tool-level). Each gate has a dedicated section in `05-signoff-criteria.md` with
explicit pass criteria.

**Time.** ~1.5 min.

### Slide E.3 — CI regression levels + waiver discipline

**Key message.** Five regression levels (L0–L4) trigger automatically; waivers are
version-controlled and required to have expiry triggers.

**On slide — two compact panels.**

*CI levels (from `04-execution-flow.md` §15):*

| Level | Trigger | Wall-clock |
|---|---|---|
| L0 | Pre-commit | < 1 min |
| L1 | Push to feature branch | < 15 min |
| L2 | PR open / update | < 30 min |
| L3 | Nightly | < 6 h |
| L4 | Release candidate | < 12 h |

*Waiver discipline:*

- Stored under `waivers/` (lint, CDC, timing).
- Required fields: ID, citation to analysis or ADR, expiry trigger.
- A waiver without an expiry trigger blocks integration.
- Reviewed at the gate it affects.

**Speaker notes.** The CI level grid is what makes the methodology *executable*
rather than aspirational. Each level builds on the previous — you can't promote
to L3 without a clean L2 at the same commit. Waiver discipline is the
anti-laundering mechanism for the implementation side: same principle as citations
on the spec side.

**Time.** ~1 min.

---

## Section F — Schedule, asks, and decisions (3 min · 2 slides)

### Slide F.1 — Schedule

**Key message.** Two milestones, six weeks total. Three named contingencies.
LLM-assisted estimate; manual baseline would be ~10–12 weeks.

**On slide — table.**

| Milestone | Outputs | Target |
|---|---|---|
| **M1** | FAD §1–§12 baselined, core ICDs frozen, ADR-0005 accepted, pilot MS drafted end-to-end | 2 weeks |
| **M2** | All MSs at `design_ready`, RTM populated, architecture officially done | +4 weeks (6 weeks total) |

Contingencies:

1. Gastón's spec template confirmation by end of week 1.
2. Source-doc open issues (OI-001, OI-S07, OI-S08) assigned owners.
3. No major architectural surprises in FAD §3–§5.

**Speaker notes.** State the LLM speedup honestly. LLMs accelerate drafting,
consistency checks, scaffolding, format work. They do not speed up engineering
thinking, ambiguity resolution, or stakeholder alignment. The 6-week estimate
assumes all three contingencies hold.

**Time.** ~1.5 min.

### Slide F.2 — Decisions to surface and asks per attendee

**Key message.** Five decisions on the table. Specific asks per attendee, with
deadlines.

**On slide — two-column layout.**

*Decisions to surface:*

| Decision | Status | Resolves with |
|---|---|---|
| ADR-0005 (spec/uArch split) | Proposed | Gastón's template confirmation |
| FAD §6 module inventory at functional-block granularity | Pending update | G1 review pass |
| Reviewer (ChatGPT project) configuration | Pending | Architect this week |
| Source-doc OIs (OI-001, OI-S07, OI-S08) | Open | Owner assignment |
| Schedule commitment (M1+M2 = 6 weeks) | Proposed | Sponsor agreement |

*Asks per attendee with suggested deadlines:*

- **Gastón** — confirm spec template, **end of week 1 (Fri 8 May 2026)**. Slack
  message saying yes, or a list of edits. Unblocks ADR-0005 and template
  replacement.
- **Shlomi** — assign owners to OI-001, OI-S07, OI-S08, **end of week 2 (Fri 15
  May 2026)**. Names + dates in the OI table. Removes contingency 2.
- **Internal team** — agree on review cadence and reviewer LLM tool, **at this
  meeting**.

**Speaker notes.** End on the asks. Be specific. Today's date is 28 April 2026.
Friday of week 1 = 8 May; Friday of week 2 = 15 May. The deadlines are *suggested*
— adjust per the actual meeting date. If the meeting slips, all dates slip
together.

**Time.** ~1.5 min.

---

## Appendix A — Decisions that don't need this meeting

- Folder rename `arch/modules/ → arch/specs/` — already rejected, kept current
  name.
- 5-doc framework (FAD/MS/ICD/ADR/RTM) — already accepted in ADR-0001.
- Spec/uArch split principle — codified in ADR-0005 (status: `proposed`); only
  *template details* are open.
- `design_ready` replacing `rtl_ready` — already done in `arch/README.md`.
- G4 split into G4a + G4b — already documented in `02-architect-workflow.md` and
  `05-signoff-criteria.md`.

(These can be cited if asked, but are not on the critical path for this meeting.)

## Appendix B — Open items the methodology surfaces but does not resolve

- Vivado / simulator / lint / CDC tool versions: pinned at first top-level
  synthesis run.
- Waiver file format (TOML / YAML / Spyglass-native): pinned when lint tool
  selected.
- CI provider and regression-level wall-clock targets: pinned before L1 first
  run.
- Hardware bring-up test plan template: pinned before first hardware delivery.

(Tracked in `04-execution-flow.md` §18 and `05-signoff-criteria.md` §19. Mention
only if asked.)

## Appendix C — Talk-track notes (general)

- Avoid framework jargon when possible. Use concrete examples (specific FAD
  section, specific module name).
- If pushed on schedule: state the LLM-assisted vs manual delta honestly. Don't
  oversell LLMs.
- If pushed on the spec/uArch split: anchor on the completeness test (D.4
  speaker notes).
- If pushed on review cadence: review at commit, not every iteration.
- If asked "why not merge framework + methodology into one doc?": separation of
  *what* vs *how* is load-bearing; spec layer must baseline independently of
  tooling choices.
- If pushed on E.1–E.3 detail (someone wants more on flow / sign-off): point to
  `04-execution-flow.md` and `05-signoff-criteria.md`; offer a follow-up dedicated
  session.

---

## Build notes (for converting this outline to pptx)

- Total slide count: 20.
- Section colour-banding: Frame (grey), Methodology (blue), Roles (green),
  Architect workflow (orange — main lane), Exec/sign-off (purple — new),
  Schedule (red).
- Reserve the largest visual real estate for D.1 (phase flow diagram) and E.2
  (gate map).
- Tables in slides D.4, D.5, D.7, D.6, E.1, E.2, E.3 — use real table objects,
  not screenshots.
- Use one canonical block diagram for B.2 (repo layout).
- Speaker notes go into the pptx notes pane verbatim.
- Cite document IDs (e.g., `02-architect-workflow.md §6.4`,
  `04-execution-flow.md §3`) on every slide footer where content is sourced.
- Compression strategy from v0.1 → v0.2: merged closely-related slides
  (D.4+D.5 → D.2; D.7+D.8 → D.5; D.10+D.11 → D.6; D.14+D.15 → D.8). Methodology
  framing compressed from 3 slides to 2 (B.1 absorbs old B.1+B.2). New section
  E added in three compact slides.

