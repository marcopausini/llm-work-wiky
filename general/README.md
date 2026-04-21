# general

Work content not bound to any specific project. Collects:

- One-off technical questions (HLS debugging, Vitis migration issues,
  tooling friction) that don't sit inside `claude-fpga-rtl-mentor` or
  `bpms-sfu-architecture`.
- Career reflections, role-transition notes, performance-review prep.
- Cross-project tooling / workflow (Claude Code setup, WSL quirks,
  Rocky Linux migration notes).
- Exploratory chats that might later graduate into a dedicated project.

## Files

- `notes.md` — working scratchpad.
- `checkpoints/` — checkpoints from the `checkpoint` skill, scope=general.
- `decisions/` — cross-cutting decisions (e.g. "switch from WSL to Rocky
  Linux for Claude Code", "adopt CLAUDE.md in every repo").

No `state.md` here — `general/` is a pool, not a project.

## Graduation trigger

When three or more checkpoints under `general/` cluster around the same
topic and a long-running thread emerges, consider promoting:

1. Add a new slug to `_meta/projects.md` under Active.
2. `mkdir projects/<new-slug>/` with the standard sub-structure.
3. Move matching checkpoints into `projects/<new-slug>/checkpoints/`.
4. Create `state.md` from accumulated content.
5. Optionally create a matching Claude Project.
