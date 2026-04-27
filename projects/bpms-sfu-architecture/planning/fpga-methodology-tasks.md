# FPGA Project Methodology — Task List

**Created:** 2026-04-26
**Author:** Marco Pausini
**Last updated:** 2026-04-27

---

## Framework and methodology

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 1 | [x] | Implement revision plan changes 1–5: patch FAD template (§1.5 invariants, §12 rename), ICD template (protocol assertions), RTM (derived req policy), module spec template (implementation evidence §12.3) | — | **Done** in commit `14bb861` (FAD/ICD/RTM) and `3eeffe0` (§12.3 carried into the new spec template) |
| 2 | [x] | Update `arch/README.md`: add execution layer reference, updated lifecycle, spec/uarch terminology | — | **Done** in commit `3eeffe0` |
| 4 | [x] | Create `constraints/README.md`: XDC file structure, naming, ownership, exception discipline | — | **Done** in commit `0e94c45` |
| 5 | [x] | Create `execution-flow.md` content: Vivado flow, CI gates, constraints/CDC methodology, IP integration policy, waiver policy, report archival | Task 4 | **Done** in commit `3a07eea` |
| 6 | [x] | Create `signoff-criteria.md` content: sign-off criteria per level, G4a/G4b split, cross-refs to flow | Task 5 | **Done** in commit `3a07eea` |
| 7 | [ ] | Update top-level repo README to reference `docs/methodology/fpga-project-methodology.md` | — | Not addressed in this session |

## Spec/uarch handoff

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 8 | [ ] | Get confirmation from Gastón on `spec.md` template: share two-level split proposal, agree on exact template sections | — | **Critical path** — external; not addressed |
| 9 | [ ] | Write and get accepted ADR-0005: spec/uarch split decision | Task 8 | **ADR drafted** in commit `3eeffe0` with `status: proposed`. Acceptance still pending Gastón's review (Task 8). Flip frontmatter to `accepted` and add a Change Log row when ready |
| 10 | [x] | Replace `arch/modules/_template.md` with adapted spec template (Gastón's template + framework conventions: frontmatter, markers, citations, system constraints, refmodel pointer) | Task 9 | **Done** in commit `3eeffe0`. Note: a generic black-box spec template was authored (no access to Gastón's). Expect revision after his review — sections, signal naming conventions, and structure may need to align with his existing flow |

## FAD and architecture content

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 11 | [ ] | Update FAD §2 block diagram and §6 module inventory to spec-level granularity (functional blocks, not .sv files). Submit to `fpga-arch-reviewer` for review. Accepted inventory drives all `arch/modules/<module_name>.md` filenames | — | Merges old tasks 8/9/20. Block partitioning is an output of this, not a separate task |
| 12 | [ ] | Define I/O plan: top-level port list, FPGA bank/pin mapping from RFX-8440A board, capture in FAD or as ICD | — | Board constrains pin assignment; architect defines I/O plan |
| 13 | [ ] | Continue FAD §3–§5 authoring (dataflow, clocking/CDC, memory) per `state.md` next-phase plan | — | |
| 14 | [ ] | Draft first module spec as pilot: pick one block and validate the template end-to-end with Gastón's RTL flow before committing to all blocks | Tasks 10, 11 | Critical for de-risking the template |

## Tool and workflow setup

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 15 | [ ] | Update Claude Project instructions: spec/uarch terminology, updated role definition, design-ready criteria | — | External (Claude Project, not in-repo) — not addressed |
| 16 | [ ] | Update `state.md`: capture all decisions from this conversation (spec/uarch split, folder structure, role ownership, execution layer) | — | External working artifact — not addressed |
| 17 | [ ] | Update `CLAUDE.md` in repo: implement `repo_operator` role (consistency checks, marker audits, RTM regeneration) — or create appropriate Claude Code skills | — | **Partial.** In-repo `CLAUDE.md` updated in commit `3eeffe0` for new spec/uarch terminology, all 8 cross-file invariants, and new prompt recipe. Repo-operator commands (consistency-check / marker-audit / RTM-regen recipes or Claude Code skills) **not** added — deferred per Step 11 of the revision plan until templates stabilise |
| 18 | [ ] | Create ChatGPT project `fpga-arch-reviewer`: instructions, required context files (framework, signoff criteria, FAD), reviewer prompt template | — | External (ChatGPT project) — not addressed |

## Communication

| # | Done | Task | Blocked by | Notes |
|---|---|---|---|---|
| 19 | [ ] | Create presentation for next meeting: full fpga-project-methodology — roles, phases, spec/uarch split, LLM workflow, gates | — | |

---

## Critical path

```text
Task 8 (Gastón confirmation)
  → Task 9 (ADR-0005 acceptance)         ← currently 'proposed'
    → Task 10 (template replacement)     ← initial replacement done; expect revision after Gastón's review
      → Task 11 (FAD §2/§6 at spec-level)
        → Task 14 (pilot spec end-to-end)
```

Everything else proceeds in parallel.

---

## Session summary (2026-04-27)

**Fully done (6/19):** 1, 2, 4, 5, 6, 10
**Partial (2/19):** 9 (ADR drafted, not accepted), 17 (CLAUDE.md terminology updated, repo-operator commands deferred)
**Not addressed (11/19):** 3, 7, 8, 11–16, 18, 19

Commits landed in this session:

| SHA | Commit |
|---|---|
| `14bb861` | `[arc](templates)`: extend FAD, ICD, RTM with framework v2 additions |
| `0e94c45` | `[arc](layout)`: add tb/, waivers/, reports/ and constraints discipline |
| `3a07eea` | `[arc](methodology)`: author execution-flow.md and signoff-criteria.md |
| `9092781` | `[arc](methodology)`: add project methodology and architect workflow drafts |
| `3eeffe0` | `[arc](0005)`: split module spec from uarch per ADR-0005 |
