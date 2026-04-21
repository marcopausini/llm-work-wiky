# llm-work-wiki

Work knowledge wiki. Decisions, analyses, checkpoints, and curated state for
FPGA/DSP engineering work and architect-track learning, produced through LLM
interaction.

Paired with Claude Projects. **No Google Drive layer** — this repo is
self-contained on the work laptop. Private remote only.

---

## What lives where

| Path | Purpose |
|---|---|
| `_inbox/` | Unfiled checkpoints. Triage when cluttered. |
| `_meta/topics.md` | Controlled tag vocabulary. Read before tagging. |
| `_meta/projects.md` | Active and dormant projects. |
| `_meta/state-template.md` | Template for `state.md` files. |
| `skills/` | Reusable Claude skills (`checkpoint`, future ones). |
| `scripts/` | Python analysis scripts. |
| `projects/<slug>/state.md` | Curated canonical state per project. Mirrored to Claude Project KB. |
| `projects/<slug>/notes.md` | Working scratchpad. |
| `projects/<slug>/checkpoints/` | Raw chat artifacts, from `checkpoint` skill. |
| `projects/<slug>/decisions/` | ADRs: Context / Decision / Consequences. |
| `general/` | Work content not bound to any specific project. Same sub-structure. |

## Active projects

- `claude-fpga-rtl-mentor` — Claude-mediated self-study for FPGA architect
  transition. Books, vendor guides, methodology frameworks.
- `bpms-sfu-architecture` — BPMS system architecture work, SFU-focused.

Full list and status: `_meta/projects.md`.

## Not here

- AST internal specs verbatim. Reference by title and revision only.
- Credentials, tokens, internal URLs, customer or partner identifiers.
- Personal-life content → `llm-life-wiki` (separate repo on personal laptop).

---

## Promotion flow

```
chat with LLM (inside a Claude Project, or not)
    │
    ▼
checkpoint skill writes → projects/<slug>/checkpoints/   or   general/checkpoints/
    │
    ▼
(triage, opportunistic)
    │
    ├── stays as checkpoint (reference only)
    ├── promoted → .../decisions/ (ADR)
    └── absorbed  → .../state.md (canonical)
                    │
                    ▼
        re-upload state.md to the corresponding Claude Project KB
```

## Inbox triage

No fixed cadence. Trigger when `_inbox/` crosses ~10 items or feels
cluttered. Monthly backstop.

## Mapping to Claude Projects

| Claude Project | Wiki path | KB file |
|---|---|---|
| claude-fpga-rtl-mentor | `projects/claude-fpga-rtl-mentor/` | `state.md` |
| bpms-sfu-architecture | `projects/bpms-sfu-architecture/` | `state.md` |
| (no Project) | `general/` | n/a — no KB mirror |

Each Claude Project's instructions should include: *"Canonical state is
`state.md` as uploaded to this Project's knowledge base. If in-chat context
contradicts it, `state.md` wins."*

## Hardware placement

- Lives on: work laptop only, `~/llm-work-wiki/`
- Remote: private GitHub (or AST-internal Git), never public.
- Personal laptop: never clones this repo.
- Mobile chat for work material: avoid; if unavoidable, output goes to
  `_inbox/` on next work-laptop session. Double-check IP rules first.
