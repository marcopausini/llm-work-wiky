---
name: checkpoint
description: Distill a chat conversation into one or more structured markdown checkpoint notes for the llm-work-wiki. Use when the user says "checkpoint", "save this chat", "checkpoint this", "distill this conversation", "create a checkpoint", or explicitly asks to save findings from the current conversation to the work wiki. Emits frontmatter-tagged markdown compliant with the wiki schema, including a structured re-seed block. Auto-detects whether the chat is bound to a Claude Project (claude-fpga-rtl-mentor, bpms-sfu-architecture) or unbound (general). Respects IP rules: no verbatim reproduction of AST internal specs; reference by title and revision only.
---

# Checkpoint Skill (work)

Distills the current chat into structured markdown checkpoint notes for the
work wiki. One chat produces one note, unless it spans distinct scopes
(different project, or project + general) — in which case emit one note per
scope.

## When to use

Trigger on any of:
- "checkpoint this", "checkpoint chat", "save checkpoint", "save this chat"
- "distill this conversation"
- "create a checkpoint" / "create a note from this"
- Explicit request to produce a wiki entry from the conversation

## When NOT to use

- User wants a plain summary — summarise inline, no wiki structure.
- User wants a single specific artifact (email, decision doc, report) —
  use appropriate skill or respond directly.
- Conversation has < ~5 substantive turns and no findings — tell the user
  there isn't enough material and ask what to preserve.

---

## Procedure

### 1. Load controlled vocabulary

**If in Claude Code with filesystem access:**
- Read `_meta/topics.md` — controlled topic tags.
- Read `_meta/projects.md` — active project slugs.

**If in Claude Chat without filesystem access:**
Use best-guess tags and flag uncertain or new ones at the end of the output.

### 2. Determine scope (project vs general)

Check, in order:

1. **System prompt / project instructions in current chat.** If the chat
   is inside a Claude Project, its name usually appears in context. Match
   against `_meta/projects.md` slugs. If matched → scope = that project.
2. **Content inference.** Is the conversation clearly about one of the
   active projects? If unambiguous → scope = that project.
3. **Fallback.** If conversation is not bound to any single active
   project → scope = `general`.

**If ambiguous (e.g. chat touches both rtl-mentor study and bpms-sfu
architecture):** ask one clarifying question:
> "This conversation touched both `claude-fpga-rtl-mentor` and
> `bpms-sfu-architecture`. Split into two checkpoints, or file under one?
> (one / rtl-mentor / bpms)"

Then proceed.

### 3. Classify note type

| Type | When |
|---|---|
| `decision` | A technical or methodological choice was made, with reasoning |
| `learning` | New information or understanding acquired (reading notes, explanation of a concept) |
| `insight` | A realisation about approach, pattern, or gap |
| `research` | Topic investigated, conclusions open-ended |
| `todo` | Primary output is action items |
| `reference` | Synthesised material for future lookup |

### 4. Write each note

Schema:

```markdown
---
id: YYYY-MM-DD--{project-or-general}--{slug}
date: YYYY-MM-DD
project: claude-fpga-rtl-mentor          # omit line if scope=general
type: decision|learning|insight|research|todo|reference
status: open|ongoing|resolved|archived
topics: [tag-1, tag-2]
source_chat: claude-opus-4.7             # or chatgpt, gemini
---

# Short descriptive title

## Context
2–3 lines. What triggered this conversation. Reference specs by title and
revision only — do not paste their content.

## Key findings
- Dense bullets. One fact or conclusion per bullet.
- Specific, not general.
- For reading notes: author's claim, then your reaction / application.

## Decisions
- What was concluded, if anything. Omit section entirely if none.

## Open questions
- What remains unresolved. Omit entirely if none.

## Action items
- [ ] Concrete next action
- [ ] ...

## Re-seed block

\```
GOAL: one-sentence objective
CONSTRAINTS:
- hard constraint (e.g. target device, Vitis version)
- hard constraint
PRIOR CONCLUSIONS:
- load-bearing technical fact from this chat
- load-bearing fact
CURRENT QUESTION: what a new chat should pick up from
\```
```

**Field rules:**

- `slug`: 2–4 kebab-case words, content-derived.
  Good: `sfu-clocking-tradeoffs`, `kilts-ch4-pipelining`, `aie-v1-buffer-pattern`.
  Bad: `notes`, `chat`, `architecture-stuff`.
- `topics`: prefer existing tags. Flag new ones.
- `project`: must match a slug in `_meta/projects.md` under Active; omit
  the line entirely if `general`.
- Omit empty sections.
- Re-seed block: strict four keys, no prose outside them.
- **IP:** reference specs by `<title> <revision>`; never paste internal
  content. If a conversation's findings depend on verbatim spec content,
  write "see BPMS 1.0 v1.3 §X.Y" and compress your own reasoning instead.

### 5. Output

**In Claude Code (filesystem available):**

- If scope is a project → write to `projects/{slug}/checkpoints/{id}.md`.
- If scope is general → write to `general/checkpoints/{id}.md`.
- If any new topic was coined, append to `_meta/topics.md` with
  `# added YYYY-MM-DD, unconfirmed`.
- Report: file path(s) written, any new tags flagged.

**In Claude Chat (no filesystem):**

- Emit each note as a fenced markdown block, preceded by a heading with
  the target filename:
  `### projects/bpms-sfu-architecture/checkpoints/2026-04-21--bpms-sfu-architecture--sfu-clocking-tradeoffs.md`
- List any new topics needing addition to `_meta/topics.md`.
- User drops output into `~/llm-work-wiki/_inbox/` on work laptop and
  files on next triage.

---

## Example — single note, project-scoped

```markdown
### projects/claude-fpga-rtl-mentor/checkpoints/2026-04-21--claude-fpga-rtl-mentor--kilts-ch4-pipelining.md

---
id: 2026-04-21--claude-fpga-rtl-mentor--kilts-ch4-pipelining
date: 2026-04-21
project: claude-fpga-rtl-mentor
type: learning
status: ongoing
topics: [kilts-book, rtl-coding, timing-closure, architect-transition]
source_chat: claude-opus-4.7
---

# Kilts Ch.4 — pipelining for timing closure, practical takeaways

## Context
Working through Kilts "Advanced FPGA Design" Ch.4. Goal: consolidate
pipelining intuition at the architect level, not just as an RTL tactic.

## Key findings
- Pipelining is a *system-level* latency/throughput knob; at architect
  layer it belongs in the ICD, not buried in RTL.
- Register balancing by synthesis is reliable only within a combinational
  cone; across hierarchy boundaries, manual stage placement wins.
- Fan-out replication is preferable to buffer trees for high-fanout
  control signals in Versal — matches what UG949 recommends for timing.

## Open questions
- How does AIE kernel latency interact with PL-side pipelining for MBF
  DFE-TX? To explore in a bpms-sfu-architecture chat.

## Action items
- [ ] Read Kilts Ch.5 (retiming) this week
- [ ] Add pipelining as explicit ICD dimension in SFU arc42 template

## Re-seed block

\```
GOAL: build architect-level intuition for pipelining decisions
CONSTRAINTS:
- Target: Versal ACAP, AIE v1 (not AIE-ML)
- Production device: xcvc1802-vsva2197-1LP
- Learning plan: 6-month architect transition
PRIOR CONCLUSIONS:
- Pipelining belongs in ICD, not just RTL
- Fan-out replication > buffer trees on Versal
- Cross-hierarchy register placement is manual
CURRENT QUESTION: how does this interact with AIE kernel latency in MBF DFE-TX?
\```
```

## Example — multi-scope split

Chat covered Kilts Ch.4 notes AND SFU clocking tradeoffs. Emit two notes:

- `projects/claude-fpga-rtl-mentor/checkpoints/...--kilts-ch4-pipelining.md`
- `projects/bpms-sfu-architecture/checkpoints/...--sfu-clocking-tradeoffs.md`

Each with its own re-seed block. The re-seed block in the RTL-mentor note
ends with a cross-reference ("→ see bpms-sfu checkpoint same date").

---

## Anti-patterns

- Do not paste AST spec content verbatim. Reference by title + revision.
- Do not emit if conversation is too thin — say so, ask.
- Do not invent project slugs or topics silently — flag them.
- Do not write preamble ("I'll create a checkpoint..."). Just emit.
- Do not include credentials, internal URLs, or partner/customer identifiers.
- Action items must be actions. "Think about X" is not. "Read UG949 §3 by
  Friday" is.
- Do not write a note longer than the signal content of the conversation.
