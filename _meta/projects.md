# Projects — work

Long-lived threads spanning multiple checkpoints and decisions. Use the
slug in frontmatter `project:` field.

Project slug ↔ folder under `projects/<slug>/` ↔ Claude Project name (when
one exists).

---

## Active

- `claude-fpga-rtl-mentor` — Claude as mentor for FPGA architect transition.
  Self-study plan covering Kilts, Maaref, Arora, Hennessy/Patterson, AMD
  vendor guides (UG949, Versal trilogy), methodology frameworks (arc42,
  ADRs, NASA GSFC).
- `bpms-sfu-architecture` — BPMS system architecture work, SFU-focused.

## Dormant

(none yet)

## Archived

(none yet)

---

## Rules

- Before creating a new project entry, confirm it's long-lived (spans
  multiple checkpoints or months). Otherwise use `general/` and tag by
  topic.
- Archiving: move the entry under Archived, and move
  `projects/<slug>/` → `projects/_archive/<slug>/`. Preserve history; do
  not delete.
- A project's Claude Project instructions should include the pointer:
  *"Canonical state is `state.md` in the project KB. If chat context
  contradicts it, `state.md` wins."*
