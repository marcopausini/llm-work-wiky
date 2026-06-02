Idea:

claude.ai as the fpga-arch
chatgpt as the fpga-arch-reviewer

Proposal:

define **separate roles and hard handoff rules**. Otherwise you just create two LLMs agreeing with each other and increasing noise.

The best setup is:

```text
Claude.ai        = primary FPGA architect / spec drafter
ChatGPT          = independent architecture reviewer / methodology checker
Claude Code      = repo-aware spec/code executor
Codex            = optional second repo-aware reviewer/executor
You             = final architect / owner / sign-off authority
```

Claude.ai and ChatGPT are enough for the **architecture documentation phase**. Claude Code and/or Codex become valuable when the work moves into the repository: consistency checks, Markdown edits, scripts, RTL scaffolding, CI, Tcl, report parsing, and eventually code review.

Claude Code can read and edit a codebase, run commands, and work across files/tools; Anthropic positions it as an agentic coding tool integrated with terminal/IDE/browser workflows. ([Claude API Docs][1]) Codex is OpenAI’s coding agent for software development; OpenAI describes Codex CLI as able to read, change, and run code locally in a selected directory, and Codex cloud as able to work in its own cloud environment. ([OpenAI Developers][2])

---

# Recommended role split

## 1. Claude.ai as `fpga_arch`

Use Claude for **drafting and iteration**.

Good tasks for Claude.ai:

```text
fill FAD sections
draft MDS files
draft ICDs
turn source notes into structured architecture text
expand rough diagrams
create ADR alternatives
extract assumptions/open questions
generate first-pass RTM rows
```

Why Claude.ai fits: it is good at long-form structured writing, iterative drafting, and staying inside a document style.

Your current workflow is sensible:

```text
source docs / your notes
→ Claude.ai drafts section
→ you inspect and correct
→ Claude.ai revises
→ output committed to Markdown
```

Keep this.

## 2. ChatGPT as `fpga_arch_reviewer`

Use ChatGPT as the **adversarial reviewer**, not as another drafter.

Good tasks for ChatGPT:

```text
find missing engineering details
check against AMD/Xilinx methodology
check FAD/MDS/ICD consistency
challenge assumptions
identify unverifiable statements
find missing CDC/constraints/timing/verification hooks
review whether an MDS is truly rtl_ready
convert review comments into actionable TODOs
```

The key is that ChatGPT should not rewrite the document first. It should produce a review report.

Preferred reviewer output:

```text
severity
location
issue
why it matters
required action
suggested text
gate impacted
```

Example:

```markdown
| Severity | Location | Issue | Action | Gate |
|---|---|---|---|---|
| High | FAD §4.5 | CDC exists from Aurora clock to DSP clock but mechanism is unspecified. | Add CDC row with async FIFO mechanism, depth, reset behavior, and owner MDS. | G1/G4/CDC |
```

## 3. Claude Code as `repo_operator`

Use Claude Code once files exist in the repo.

Good tasks:

```text
apply template changes across files
update links and frontmatter
create new MDS files from module inventory
run grep consistency checks
generate doc_check scripts
generate Makefile/Tcl skeletons
summarize Vivado reports
patch Markdown after your approval
```

Claude Code should operate on the actual repo. That is the main difference versus Claude.ai.

For example:

```text
“Scan arch/modules/*.md and list all MDSs marked rtl_ready that still contain TBD/STUB.”
```

or:

```text
“Create docs/methodology/flow.md and signoff.md from this template. Do not modify FAD or MDS files yet.”
```

Anthropic’s Claude Code docs also describe project memory through `CLAUDE.md`, plus project/user settings and hooks, which fits your desired repeatable workflow. ([Claude API Docs][3])

## 4. Codex as optional `repo_reviewer` or `script_executor`

Do not add Codex just because it exists. Add it if you want an independent repo-aware agent, especially for:

```text
reviewing Claude Code patches
creating Python/Tcl/doc_check scripts
checking CI workflows
summarizing diffs
reviewing generated RTL or testbench scaffolding
running local code tasks in a separate toolchain
```

A good role for Codex:

```text
Codex = independent code/process reviewer
Claude Code = primary repo editor
```

or the reverse. Do not let both edit the same branch unsupervised.

---

# Recommended setup

## Use four named roles

Create four role prompts/files.

```text
roles/
├── fpga_arch.md
├── fpga_arch_reviewer.md
├── repo_operator.md
└── repo_reviewer.md
```

### `fpga_arch.md` — Claude.ai

Purpose:

```text
Draft architecture documents from source notes and user decisions.
```

Core rules:

```text
- Do not invent requirements.
- Mark inferred content with [INFERRED].
- Mark assumptions with [ASSUMPTION: ..., expiry trigger].
- Preserve specified / inferred / proposed separation.
- Use FAD/MDS/ICD/ADR/RTM templates.
- Produce Markdown ready to paste into the repo.
- Prefer small section-level outputs, not entire documents at once.
```

### `fpga_arch_reviewer.md` — ChatGPT

Purpose:

```text
Review architecture documents for completeness, consistency, AMD/Xilinx methodology alignment, and RTL-readiness.
```

Core rules:

```text
- Do not rewrite unless asked.
- Produce issues, not prose commentary.
- Classify severity: blocker / high / medium / low.
- Tie each issue to a concrete action.
- Check against FAD/MDS/ICD/RTM consistency.
- Check constraints, CDC, reset, timing, verification, debug, sign-off evidence.
- Challenge assumptions.
- Identify unverifiable or uncited claims.
```

### `repo_operator.md` — Claude Code

Purpose:

```text
Modify repository files, run checks, and keep the Markdown/spec tree consistent.
```

Core rules:

```text
- Work on one branch.
- Make minimal diffs.
- Do not change frozen/baselined docs without explicit instruction.
- Run available checks after edits.
- Summarize changed files.
- Do not silently resolve TBD/ASSUMPTION markers.
```

### `repo_reviewer.md` — Codex

Purpose:

```text
Review repository changes and generate scripts/tests/checkers.
```

Core rules:

```text
- Review diffs against methodology.
- Flag mismatches between spec and implementation.
- Prefer scripts and automated checks over manual comments.
- Do not make architecture decisions.
```

---

# Practical workflow

## Phase A — architecture drafting

Use Claude.ai.

```text
Input:
- relevant parent doc section
- your notes
- target template section

Output:
- one FAD/MDS/ICD section
- unresolved items table
- assumptions/inferences clearly marked
```

Do not ask Claude to fill the whole FAD at once. Your current section-by-section approach is correct.

## Phase B — independent review

Paste the drafted section into ChatGPT with a reviewer prompt.

Use this prompt:

```text
Act as fpga_arch_reviewer.

Review the following SFU FPGA architecture section.

Check for:
1. missing requirements or derived requirements
2. unspecified clock/reset/CDC behavior
3. ambiguous interfaces
4. missing constraints/timing implications
5. missing verification hooks
6. missing execution evidence
7. assumptions incorrectly presented as facts
8. AMD/Xilinx methodology gaps

Output only:
- blocker/high/medium/low issues
- concrete action
- suggested replacement text where useful
- affected gate: FAD/MDS/ICD/RTM/flow/signoff
```

## Phase C — controlled merge

You decide which comments are valid.

Then either:

```text
manual edit
```

or:

```text
Claude Code applies the accepted changes to the repo
```

Do not let ChatGPT directly become the second drafter unless the section is clearly weak.

## Phase D — repo consistency checks

Use Claude Code.

Examples:

```text
Scan all MDS files and find:
- missing source_docs
- rtl_ready with non-empty rtl_ready_blocking
- TBD without owner
- ASSUMPTION without expiry trigger
- ICD links that do not exist
```

Then:

```text
Create or update docs/methodology/flow.md and signoff.md using the agreed templates.
```

## Phase E — optional Codex review

Use Codex after Claude Code makes repo changes.

Good Codex task:

```text
Review this branch against docs/methodology/signoff.md.
Find missing execution gates, broken links, inconsistent frontmatter, and any Markdown sections that violate the methodology.
Do not edit files. Produce a review report.
```

Or:

```text
Implement scripts/doc_check.py according to docs/methodology/flow.md.
Run it on arch/ and report failures.
```

---

# Recommended tool usage by task

| Task                              | Best tool                      |
| --------------------------------- | ------------------------------ |
| First draft of FAD/MDS prose      | Claude.ai                      |
| Critical methodology review       | ChatGPT                        |
| Cross-document consistency review | ChatGPT or Claude Code         |
| Bulk Markdown edits in repo       | Claude Code                    |
| Generate `doc_check.py`           | Claude Code or Codex           |
| Review generated scripts          | Codex or ChatGPT               |
| Generate RTL skeleton from MDS    | Claude Code or Codex           |
| Review RTL against MDS            | ChatGPT/Codex                  |
| Run tests / scripts               | Claude Code or Codex           |
| Summarize Vivado reports          | Claude Code, Codex, or ChatGPT |
| Final decision                    | you                            |

---

# Should ChatGPT draft instead of Claude?

Yes, but only selectively.

Use ChatGPT to draft when:

```text
you want a more skeptical or methodology-driven version
Claude produced vague prose
the section is about sign-off, constraints, CDC, verification, or execution gates
you need a compact engineering checklist
```

Use Claude to draft when:

```text
you are expanding architecture narrative
you want consistent Markdown style
you are iterating from source notes
you are producing long document sections
```

The strongest pattern is:

```text
Claude drafts.
ChatGPT reviews.
You decide.
Claude Code applies.
Codex optionally checks/scripts.
```

---

# Minimum setup I would start with

Do not start with all four tools at full complexity.

Start with this:

```text
1. Claude.ai drafts FAD/MDS/ICD sections.
2. ChatGPT reviews each section using fpga_arch_reviewer prompt.
3. You accept/reject review items.
4. Claude Code applies accepted edits to repo.
```

Add Codex later for:

```text
doc_check.py
Makefile/Tcl flow
Vivado report parsing
independent branch review
RTL/testbench work
```

---

# Concrete folder setup

In the repo:

```text
bpms-sfu-fpga-design/
├── arch/
├── docs/
│   ├── methodology/
│   │   ├── flow.md
│   │   └── signoff.md
│   └── ai_workflow/
│       ├── roles/
│       │   ├── fpga_arch.md
│       │   ├── fpga_arch_reviewer.md
│       │   ├── repo_operator.md
│       │   └── repo_reviewer.md
│       ├── prompts/
│       │   ├── draft_fad_section.md
│       │   ├── draft_mds.md
│       │   ├── review_arch_doc.md
│       │   ├── review_rtl_ready.md
│       │   └── apply_review_items.md
│       └── logs/
│           └── README.md
```

I would commit `roles/` and `prompts/`.

I would not commit raw chat logs unless they contain durable decisions. Instead, convert them into ADRs, FAD text, MDS text, or review issue tables.

---

# The main risk

The main risk is **LLM laundering**:

```text
Claude invents or over-assumes.
ChatGPT reviews based on the same incomplete context.
You get a polished but still wrong architecture.
```

Countermeasure:

```text
Every factual statement needs source citation, [INFERRED], or [ASSUMPTION].
Every architecture decision needs ADR or FAD decision row.
Every “ready” state needs execution evidence.
```

That is why your placeholder/citation discipline is critical.

---

# Final recommendation

Use this setup:

```text
Claude.ai        primary drafter
ChatGPT          independent reviewer
Claude Code      repo editor/check runner
Codex            optional independent repo reviewer/script generator
```

Do not use all tools at every step. Use them as a pipeline with clear boundaries.

The useful loop is:

```text
Draft → Review → Decide → Apply → Check → Sign off
```

Not:

```text
Ask many models until one answer feels convincing
```

[1]: https://docs.anthropic.com/en/docs/claude-code/overview?utm_source=chatgpt.com "Claude Code overview - Claude Code Docs"
[2]: https://developers.openai.com/codex?utm_source=chatgpt.com "Codex | OpenAI Developers"
[3]: https://docs.anthropic.com/en/docs/claude-code/memory?utm_source=chatgpt.com "How Claude remembers your project - Claude Code Docs"
