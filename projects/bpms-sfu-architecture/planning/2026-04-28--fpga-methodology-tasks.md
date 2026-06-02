# FPGA Project Methodology — Task List

**Created:** 2026-04-26
**Author:** Marco Pausini
**Last updated:** 2026-04-28

---

## Framework and methodology

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 1 | [x] | Implement revision plan changes 1–5: patch FAD template (§1.5 invariants, §12 rename), ICD template (protocol assertions), RTM (derived req policy), module spec template (implementation evidence §12.3) | — | **Done** in commit `14bb861` (FAD/ICD/RTM) and `3eeffe0` (§12.3 carried into the new spec template) |
| 2 | [x] | Update `arch/README.md`: add execution layer reference, updated lifecycle, spec/uarch terminology | — | **Done** in commit `3eeffe0` |
| 4 | [x] | Create `constraints/README.md`: XDC file structure, naming, ownership, exception discipline | — | **Done** in commit `0e94c45` |
| 5 | [x] | Create `execution-flow.md` content: Vivado flow, CI gates, constraints/CDC methodology, IP integration policy, waiver policy, report archival | Task 4 | **Done** in commit `3a07eea`. Further updated to v0.3 in session 2026-04-28: CDC philosophy locked (FAD §4.5 = architecture-level policy only; Vivado does per-flop enumeration); MDS→MS, rtl_ready→design_ready, §11.7→named section — 14 fixes across the file. |
| 6 | [x] | Create `signoff-criteria.md` content: sign-off criteria per level, G4a/G4b split, cross-refs to flow | Task 5 | **Done** in commit `3a07eea`. **Pending:** stale terminology sweep (MDS→MS, rtl_ready→design_ready) not yet done — flagged 2026-04-28. |
| 7 | [x] | Update top-level repo README to reference `docs/methodology/fpga-project-methodology.md` | — | **Done** 2026-04-28. Added `## Methodology` section with 5-row table, `docs/methodology/` to layout tree and folder table, `waivers/` to tree, MDS→MSs fix. |

## Spec/uarch handoff

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 8 | [x] | Get confirmation from Gastón on `spec.md` template: share two-level split proposal, agree on exact template sections | — | **Template ready to share (2026-04-28).** Annotated version (`_template_annotated.md`) carries ARCH-ADDITION markers and delta-summary for his review. Clean version (`_template_clean.md`) is the repo-commit version. 6 architect-side additions on Gastón's ip-specs.md baseline; CDC mechanism sub-section removed (lives in FAD §4.5 only). See checkpoint 2026-04-28. |
| 9 | [x] | Write and get accepted ADR-0005: spec/uarch split decision | Task 8 | **ADR drafted** in commit `3eeffe0` with `status: proposed`. Acceptance pending Gastón's review (Task 8). |
| 10 | [x] | Replace `arch/modules/_template.md` with adapted spec template | Task 9 | **Done 2026-04-28.** Clean production version finalised (214 lines, no scaffolding). Annotated version retained for review handoff. Pending: commit to repo once Gastón confirms (Task 8). |

## FAD and architecture content

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 11 | [ ] | Update FAD §2 block diagram and §6 module inventory to spec-level granularity (functional blocks, not .sv files). Submit to `fpga-arch-reviewer` for review. Accepted inventory drives all `arch/modules/<module_name>.md` filenames | Tasks 8, 9 | Merges old tasks 8/9/20. Block partitioning is an output of this, not a separate task. |
| 12 | [ ] | Define I/O plan: top-level port list, FPGA bank/pin mapping from RFX-8440A board, capture in FAD or as ICD | — | Board constrains pin assignment; architect defines I/O plan |
| 13 | [ ] | Continue FAD §3–§5 authoring (dataflow, clocking/CDC, memory) per `state.md` next-phase plan | — | Note: when ADR-0002 (clock architecture) is drafted, record that FAD §4.5 scope is architecture-level only (clock relationships, approved mechanisms, intentional crossings); per-flop enumeration is Vivado's job. |
| 14 | [ ] | Draft first module spec as pilot: pick one block and validate the template end-to-end with Gastón's RTL flow before committing to all blocks | Tasks 10, 11 | Critical for de-risking the template |

## Tool and workflow setup

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 15 | [x] | Update Claude Project instructions: spec/uarch terminology, updated role definition, design-ready criteria | — | External (Claude Project, not in-repo) — not addressed |
| 16 | [x] | Update `state.md`: capture all decisions from sessions 2026-04-26 and 2026-04-28 | — | Pending |
| 17 | [x] | Update `CLAUDE.md` in repo: implement `repo_operator` role (consistency checks, marker audits, RTM regeneration) | — | **Partial.** In-repo `CLAUDE.md` updated in commit `3eeffe0` for spec/uarch terminology. Repo-operator commands deferred until templates stabilise. |
| 18 | [ ] | Create ChatGPT project `fpga-arch-reviewer`: instructions, required context files (framework, signoff criteria, FAD), reviewer prompt template | — | External (ChatGPT project) — not addressed |

## Pending cleanup

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 20 | [x] | Sweep `05-signoff-criteria.md` for stale terminology: MDS→MS, rtl_ready→design_ready, rtl_ready_blocking→design_ready_blocking, any §N.N cross-refs to old template sections | — | Flagged 2026-04-28. Same pass as execution-flow v0.3 but not yet done. |
| 21 | [x] | Commit all artefacts from 2026-04-28 session: `_template_clean.md` (as `arch/modules/_template.md`), `04-execution-flow.md` v0.3, `waivers/README.md`, `README.md` | Task 8 (template); others unblocked | Template commit waits for Gastón confirmation; others can land now. |
| 22 | [x] | Author `waivers/README.md` operational section: one worked example per domain (lint, CDC, timing) with concrete file-format entry | EXEC-FLOW-001/004 (lint tool pinned) | Policy section complete; format examples deferred until tool baseline pinned. |
| 23 | [x] | Review project folder layout in `bpms-sfu-fpga-design`: evaluate supporting more than one Vivado `prj/` and moving `constraints/` under each `prj/`. Output a written recommendation (ADR if decision-grade) — no file moves in this task. | — | Review/decision only. Touches design-repo structure, not the wiki. |
| 24 | [x] | Update top-level repo README to add a prominent pointer naming `arch/README.md` as "the single source of truth for the SFU FPGA design". | — | Follow-up to Task 7 (which added the `## Methodology` section). Short, surgical edit — likely one line near the top of the README. |

## Communication

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 19 | [x] | Create presentation for next meeting: full fpga-project-methodology — roles, phases, spec/uarch split, LLM workflow, gates | — | |

---

## Critical path

```text
Task 8 (Gastón confirmation)
  → Task 9 (ADR-0005 acceptance)
    → Task 10 commit (template to repo)
      → Task 11 (FAD §2/§6 at spec-level)
        → Task 14 (pilot spec end-to-end)
```

Unblocked parallel work: Task 20 (signoff-criteria sweep), Task 21 (commit non-template artefacts), Task 13 (FAD §3–§5), Task 16 (state.md update).

---

## Session summary (2026-04-28)

**Fully done (new):** 7, 10 (production artefact complete; commit pending Task 8)
**Updated:** 5 (execution-flow v0.3 — CDC philosophy + 14 terminology fixes)
**New tasks added:** 20 (signoff-criteria sweep), 21 (batch commit), 22 (waivers examples)
**Still pending (critical path):** 8, 9, 11, 14

Artefacts produced this session:

| Artefact | Target path |
|---|---|
| `_template_clean.md` | `arch/modules/_template.md` |
| `_template.md` (annotated) | `projects/bpms-sfu-architecture/planning/_template_annotated.md` |
| `04-execution-flow.md` v0.3 | `docs/methodology/04-execution-flow.md` |
| `waivers-README.md` | `waivers/README.md` |
| `top-README.md` | `README.md` (repo root) |

Previous session (2026-04-27):

| SHA | Commit |
|---|---|
| `14bb861` | `[arc](templates)`: extend FAD, ICD, RTM with framework v2 additions |
| `0e94c45` | `[arc](layout)`: add tb/, waivers/, reports/ and constraints discipline |
| `3a07eea` | `[arc](methodology)`: author execution-flow.md and signoff-criteria.md |
| `9092781` | `[arc](methodology)`: add project methodology and architect workflow drafts |
| `3eeffe0` | `[arc](0005)`: split module spec from uarch per ADR-0005 |