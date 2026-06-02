# llm-work-wiki — root agent rules

Work knowledge wiki. Contents: decisions, notes, checkpoints, and curated
project state produced through interaction with LLM tools in the context of
Marco's FPGA/DSP engineering work at AST SpaceMobile.

Paired with Claude Projects (active chat surface). 

## Confidentiality and IP posture

This repo lives on the work laptop, in a private remote only.

- **Never commit credentials, tokens, API keys, account numbers, internal
  URLs, or VPN endpoints.**
- **Never reproduce AST internal specs verbatim.** Reference by title and
  revision (e.g. "BPMS 1.0 spec v1.3, Shlomi Kulik"); paraphrase the
  substance; never paste sections.
- **Never store customer-identifying or partner-identifying material.**
- Commit messages must not leak sensitive substance. Prefer
  `checkpoint(bpms): architecture review` over explicit product details
  that would not belong on a public status page.
- **Remote:** private GitHub repo, or AST-internal Git if available and
  policy permits. Never a public remote.
- **If in doubt about whether a piece of content belongs here, default to
  not storing it.** The wiki is useful at 50% content; a leak makes it
  unusable at any percentage.

## Repo shape

See `README.md` for the canonical folder map. Two primary buckets:

- `projects/<slug>/` — content bound to a specific Claude Project.
- `general/` — work content not tied to any single project.

## Project state files

Canonical state snapshots (`state.md`) live in `projects/<slug>/state/` with a date prefix matching the format `YYYY-MM-DD--state.md`. This makes state a versioned, enumerated artefact with a snapshot per significant milestone. When triaging inbox items, rename any `state.md` with its snapshot date before filing to the `state/` folder.

## Controlled vocabulary — read before writing

Before tagging any new note:

1. Read `_meta/topics.md` — controlled topic tags.
2. Read `_meta/projects.md` — active project slugs.
3. If a new tag is needed, append with `# added YYYY-MM-DD, unconfirmed`.

Never invent tags silently. Tag drift kills retrieval.

## Writing rules

- Smallest possible change. Do not reorganise files unless asked.
- No placeholders, no `TODO: fill in later` in generated content.
- Chain-of-thought before implementation for anything non-trivial.
- Never guess. If context is missing, ask.
- Dense, technical, no hedging. Match the user's engineering register.

## Commit hygiene

Format: `{type}({scope}): {short description}`

Where `{scope}` is a project slug or `general`:

- `checkpoint(bpms): sfu interface analysis`
- `checkpoint(rtl-mentor): kilts chapter 4 notes`
- `checkpoint(general): vitis 2024 workflow migration`
- `decision(bpms): clocking strategy for SFU core`
- `state(rtl-mentor): monthly refresh`
