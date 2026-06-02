---
id: 2026-04-29--bpms-sfu-architecture--repo-operator-skill
date: 2026-04-29
project: bpms-sfu-architecture
type: decision
status: ongoing
topics: [repo-operator, claude-code, skill, consistency-check, legacy-term-audit, task-17, methodology]
source_chat: claude-sonnet-4-6
---

# Task 17 complete: repo-operator Claude Code skill

## Context

Task 17 in `fpga-methodology-tasks.md` was partially done after 2026-04-27 session —
CLAUDE.md had new terminology and invariants but lacked the actual `repo_operator`
recipe set. Remaining work: author all consistency/audit/regeneration recipes as
a Claude Code skill per `02-architect-workflow.md` §6.7.

## Key findings

- Skill form (`.claude/skills/repo-operator/`) chosen over CLAUDE.md sections:
  shared role definition lives once in SKILL.md; progressive disclosure loads only
  the relevant recipe file per invocation; recipes can be extended without touching
  CLAUDE.md.
- Umbrella `consistency-check` chains the six read-only audits in sequence;
  `rtm-regenerate` and `template-scaffold` are write-capable and excluded from
  the umbrella — explicit invocation only.
- Two grep helpers (`scripts/grep_markers.sh`, `scripts/grep_legacy.sh`) are
  read-only, idempotent, intentionally over-match; severity classification lives
  in the recipe, not the script.
- **First pilot run produced 11 blockers, 9 warnings, 5 info.** Blocker breakdown:
  legacy terms (`MDS`, `rtl_ready`) surviving in `arch/fad/_template.md`,
  `docs/methodology/03-rtl-design-workflow.md`, `docs/methodology/05-signoff-criteria.md`;
  missing frontmatter fields in 4 documents; 1 malformed `[TBD]` without owner.
  Claude Code offered to apply the mechanical sweep as one batch diff.
- `module-inventory-sync` rule 3 will flood during Task 11 transition (31 → ~10
  blocks): recommended fix is a transitional severity clause in the recipe body
  citing Task 11 as the expiry trigger, downgrading from `blocker` to `info` until
  Task 11 closes.
- `rtm-regenerate` carries a stability caveat: depends on MS §1 traceability table
  format, which is not locked until Task 14 (pilot spec). Must not be auto-applied
  until then.
- `template-scaffold` carries a stability caveat: depends on
  `arch/modules/_template.md`, which may revise after Gastón review (Task 8 / 9).

## Decisions

- Skills form adopted (option a) over CLAUDE.md prose sections — more durable,
  progressive disclosure, one skill file per recipe.
- Umbrella excludes write-capable recipes (rtm-regenerate, template-scaffold) to
  prevent accidental mutations from "audit the repo" invocations.
- Severity policy: `blocker` = invariant violation; `warning` = engineer judgement
  needed; `info` = awareness only. Consistent across all nine recipes.
- Every recipe finding cites its source rule (CLAUDE.md inv. N, arch/README.md §X,
  methodology doc §Y) — anti-laundering discipline applied to the tooling itself.
- Legacy-term sweep covers: `MDS`, `MDS-time`, `MDSs`, `rtl_ready`,
  `rtl_ready_blocking`. ADR-0005 source file excluded by the grep helper.

## Open questions

- Whether Task 11 (FAD §6 31→~10 migration) produces enough noise during transition
  to warrant the transitional severity clause or whether raising the umbrella
  stop-rule threshold is sufficient — pilot run will determine this.
- `rtm-regenerate` accuracy against actual MS §1 format: not validated until
  Task 14 produces a real module spec with a filled §1 traceability table.

## Action items

- [ ] Apply the mechanical PR offered by the consistency-check pilot: legacy-term
  sweep across 3 files + 4 frontmatter fixes + 1 malformed TBD fix.
- [ ] Add transitional severity clause to `module-inventory-sync` rule 3 if Task 11
  noise is too high after the first sweep. Expiry trigger: Task 11 closed in state.md.
- [ ] Re-run `consistency-check` after the mechanical PR to confirm clean (0 blockers).
- [ ] Commit the skill per the suggested commit message in the delivery chat.
- [ ] Mark Task 17 as fully done in `fpga-methodology-tasks.md`.

## Artefacts produced

- `.claude/skills/repo-operator/SKILL.md` — skill orchestrator; recipe dispatch table; invocation intent map
- `.claude/skills/repo-operator/references/consistency-check.md` — umbrella recipe
- `.claude/skills/repo-operator/references/marker-audit.md` — [TBD]/[STUB]/[ASSUMPTION]/[INFERRED] grammar check
- `.claude/skills/repo-operator/references/frontmatter-validate.md` — per-doc-type YAML schema check
- `.claude/skills/repo-operator/references/cdc-inventory-sync.md` — FAD §4.5 ↔ MS §6.3 sync
- `.claude/skills/repo-operator/references/module-inventory-sync.md` — FAD §2 ↔ §6 ↔ arch/modules/ ↔ rtl/ chain
- `.claude/skills/repo-operator/references/citation-audit.md` — heuristic uncited-claim sweep
- `.claude/skills/repo-operator/references/legacy-term-audit.md` — retired-term migration sweep
- `.claude/skills/repo-operator/references/rtm-regenerate.md` — RTM diff generation (⚠ caveat: Task 14)
- `.claude/skills/repo-operator/references/template-scaffold.md` — MS/ICD/ADR skeleton creation (⚠ caveat: Task 9)
- `.claude/skills/repo-operator/scripts/grep_markers.sh` — read-only marker grep helper
- `.claude/skills/repo-operator/scripts/grep_legacy.sh` — read-only legacy-term grep helper

## Re-seed block

GOAL: Apply the consistency-check pilot mechanical PR, then proceed to Task 11 (FAD §6 update to ~10 functional blocks) and Task 14 (pilot module spec end-to-end).
CONSTRAINTS:
- Target device: BittWare RFX-8440A (AMD XCZU43DR-2FFVE1156E)
- Source of truth: BPMS-1.0-SFU-001 v1.6 (primary), BPMS-1.0-ARCH-001 v2.4 (secondary)
- repo-operator skill installed at `.claude/skills/repo-operator/`; CLAUDE.md invariants unchanged
PRIOR CONCLUSIONS:
- Task 17 fully done: skill at `.claude/skills/repo-operator/` with 9 recipes + 2 helpers
- First consistency-check run: 11 blockers (mechanical; Claude Code offered single-batch fix)
- rtm-regenerate and template-scaffold carry stability caveats tied to Task 14 and Task 9 respectively
- module-inventory-sync rule 3 will flood during Task 11 transition; transitional severity clause is the clean fix
CURRENT QUESTION: Apply the mechanical PR from the consistency-check pilot (legacy terms + frontmatter + malformed TBD), re-run consistency-check to confirm clean, then start Task 11 (FAD §6 at spec-level granularity).
