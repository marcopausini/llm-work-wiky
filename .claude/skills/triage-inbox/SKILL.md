---
name: triage-inbox
description: "Triage files in the llm-work-wiki _inbox/ folder. Classify each as checkpoint or artefact, resolve target paths from checkpoint Artefacts-produced sections, validate frontmatter tags against _meta/, propose moves as a table, wait for explicit user approval, then execute git mv. Trigger phrases: triage inbox, file inbox, process my inbox, sort wiki notes."
disable-model-invocation: true
allowed-tools: Read Glob Grep Bash(git mv *) Bash(mv *) Bash(ls *) Bash(mkdir -p *) Bash(git status *) Bash(git ls-files *)
argument-hint: "[optional: glob pattern, e.g. 2026-04-*]"
---

# Triage inbox

Triage files in the llm-work-wiki `_inbox/` folder, propose moves to their correct destinations, and execute on user approval. This skill assumes the wiki is git-tracked.

## Wiki layout (reference)

```
<wiki-root>/
├── _inbox/                                # files awaiting triage (input)
├── _meta/
│   ├── topics.md                          # confirmed topic vocabulary
│   └── projects.md                        # active project slugs
├── general/
│   └── checkpoints/                       # general-scope checkpoints
└── projects/
    ├── <project-slug>/
    │   ├── checkpoints/                   # project-scoped checkpoints
    │   ├── decisions/                     # ADRs, decision records
    │   └── ... (other content per project conventions)
    └── ...
```

## File classification

A file is a **checkpoint** if its YAML frontmatter contains an `id:` field matching the pattern `YYYY-MM-DD--<scope>--<slug>` AND a `type:` field with one of `decision | learning | insight | research | todo | reference`.

Anything else is an **artefact**: block specs, ADRs, FAD/MDS/ICD documents, review notes, templates, etc.

## Phase 1 — Inventory

1. Run `ls _inbox/` to list current contents. If `$ARGUMENTS` is non-empty, treat it as a glob pattern and filter the listing.
2. If the inbox is empty (after any filter), report "Inbox empty — nothing to triage" and stop. Do not proceed.
3. Report the count of files found before continuing.

## Phase 2 — Classify each file

For every file in the filtered list:

1. Read the file's YAML frontmatter (lines between the first `---` markers).
2. Apply the classification rule above. Record one of: `checkpoint`, `artefact`, or `unclassified` (no frontmatter, malformed frontmatter, or fields missing).
3. For `checkpoint` files, capture from frontmatter: `id`, `project` (may be absent → general scope), `type`, `topics` list, `source_chat`.
4. For `checkpoint` files, also capture the `## Artefacts produced` section if present. Each bullet is expected to name a target path. Parse those paths — they are the source of truth for sibling artefact destinations.

## Phase 3 — Resolve destinations

Apply these rules in order. First match wins.

| File class | Frontmatter signal | Destination |
|---|---|---|
| Checkpoint with `project: <slug>` | `project` field present and non-empty | `projects/<slug>/checkpoints/<filename>` |
| Checkpoint without `project:` field | line absent or empty | `general/checkpoints/<filename>` |
| Artefact named in a sibling checkpoint's Artefacts-produced section | filename or basename appears in the bullet list | use the target path from that bullet, verbatim |
| Artefact NOT named in any sibling checkpoint | no match in any inbox checkpoint | flag as `needs-manual-routing`; propose nothing |
| Unclassified | no frontmatter or malformed | flag as `unclassified`; propose nothing |

A "sibling checkpoint" is one in the same triage batch (the current `_inbox/` filtered list). Do NOT scan checkpoints already filed under `projects/` or `general/` — only the current batch is consulted.

If a destination directory does not yet exist, plan a `mkdir -p` for it during execution. Note this in the proposed-moves table.

## Phase 4 — Validate tags

1. Read `_meta/topics.md`. Extract confirmed topic tags (treat any kebab-case token at start of a list item or table row as a confirmed tag).
2. Read `_meta/projects.md`. Extract confirmed project slugs the same way.
3. For each checkpoint, compare its `topics` list against the confirmed topics. Any topic NOT in `_meta/topics.md` is `unconfirmed`.
4. For each checkpoint, compare its `project` field (if present) against confirmed projects. A `project` slug not in `_meta/projects.md` is `unconfirmed-project`.

If `_meta/topics.md` or `_meta/projects.md` does not exist, treat all tags and slugs as unconfirmed and note the missing meta file in the report.

## Phase 5 — Propose moves

Emit a single Markdown table with these exact columns, in this order:

| # | Source | Destination | Class | Reason | Flags |
|---|---|---|---|---|---|

Column rules:

- **Source**: relative path from wiki root, e.g. `_inbox/2026-04-25--bpms-sfu-architecture--fad-section-2-and-6-bootstrap.md`.
- **Destination**: relative path from wiki root. Use `[NEEDS DIR CREATE]` suffix on the path if a `mkdir -p` is required.
- **Class**: `checkpoint` | `artefact` | `unclassified`.
- **Reason**: short phrase. For checkpoints: cite the rule (`project=<slug>` or `no project → general`). For artefacts: cite the source checkpoint and bullet (`from <checkpoint-id> Artefacts-produced #<n>`). For unclassified or unrouted: `needs-manual-routing` or `unclassified`.
- **Flags**: comma-separated list of any of: `unconfirmed-tag:<tag>`, `unconfirmed-project:<slug>`, `meta-missing:topics`, `meta-missing:projects`, `needs-manual-routing`, `unclassified`. Empty if none apply.

After the table, list any unrouted or unclassified files separately under a `## Unresolved` heading with one line per file explaining what's missing.

## Phase 6 — HARD STOP

After emitting the table:

```
STOP. Do not execute any moves yet.

Reply with:
  "execute"            — run all proposed moves as shown
  "execute except N"   — run all moves except row N (and any others listed)
  "cancel"             — abort, no changes
  <revised plan>       — corrected table; re-validate and re-propose
```

Do not call any `mv` or `git mv` until the user replies with `execute` or `execute except ...`. Asking the user to confirm is the entire purpose of this phase. Treat any other response as a revision — re-run from Phase 3 with the user's input as constraint.

## Phase 7 — Execute

Only on `execute` or `execute except N`:

1. Confirm wiki is git-tracked: `git status` should succeed. If not, abort with a clear error.
2. For each row to execute (skipping any explicitly excepted):
   1. If destination directory has `[NEEDS DIR CREATE]`, run `mkdir -p <dir>` first.
   2. Run `git mv <source> <destination>`.
   3. On error, stop the batch immediately. Report which row failed and why. Do not proceed with remaining rows.
3. After all moves complete, run `git status --short` and report the staged changes.
4. Do NOT commit. Per the wiki convention, the user reviews `git status` and commits manually.

Report a final summary: `<N> files moved, <M> directories created, <K> rows skipped`.

## Edge cases

- **Empty filter**: argument like `2099-*` matches nothing — Phase 1 handles this and exits cleanly.
- **Duplicate destination**: two source files resolve to the same destination → flag as `collision`, do not auto-resolve, defer to user.
- **Symlinks in `_inbox/`**: treat as regular files for classification but warn in the proposal.
- **Files without `.md` extension**: classify as `artefact` with `Class: artefact (non-md)` and let user route manually.
- **`_meta/` files mistakenly dropped in `_inbox/`**: detect by filename match against `topics.md`, `projects.md`. Propose moving to `_meta/` directly. Flag as `meta-file-misfiled`.

## Output discipline

- One table per triage run. No partial tables, no streaming output.
- No "I'll now process..." preamble. Emit the inventory count, then go silent until the table is ready.
- After the STOP block, do not add commentary, suggestions, or follow-ups. The next turn belongs to the user.
