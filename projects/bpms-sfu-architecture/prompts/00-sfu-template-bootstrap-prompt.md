# Starter prompt — paste into your SFU Claude Project

---

I uploaded `fpga-architect-resource-guide.md` in the project files — a curated guide of FPGA architecture resources (books, vendor guides, spec frameworks, decision templates).

Before we write any SFU block spec, I want to establish the full document set and templates that an FPGA architect should produce for a project like BPMS 1.0 / SFU. The block spec template in our project instructions is one deliverable — but it's not the only one.

## Step 1 — Identify the full document set

Cross-reference the roadmap's specification frameworks (NASA GSFC methodology, arc42, MADRs, Wipro "Designed-for-FPGA", UG949 checklist) against our SFU project scope and deliverable pipeline.

Propose a complete, numbered list of document types that this project should produce. For each document:

- name and filename (following our snake_case convention)
- purpose: what decision or communication does it serve?
- audience: who reads it and when?
- upstream input: what feeds into it?
- downstream consumer: what depends on it?
- source framework: which methodology inspired it?

Think beyond block specs. Consider: architecture overview, functional boundary definition, interface control documents, clock/reset/CDC strategy, memory and resource budget, timing budget, design review checklists, ADR log, assumptions log, open questions log, verification strategy, debug/telemetry plan, floorplan intent.

Don't invent documents for completeness — only propose what adds real value for our pipeline: `system docs → .md architecture/block specs → claude-code → SystemVerilog RTL`.

## Step 2 — Identify which external resources I should upload

For each document template you propose, tell me which specific external resource (and which section/chapter) would most improve the template quality if I uploaded it. Be precise — not "upload UG949" but "upload UG949 Chapter 2: Design Planning and Device Resource Analysis (pp. 15–48)."

Rank them by impact. I'll upload the top 3–5 in a follow-up message.

## Step 3 — Create the templates

Without waiting for the uploads (we'll refine later), produce a first-pass version of every template as a single consolidated Markdown file: `sfu_document_templates.md`.

Each template should include:

- all section headings with brief guidance on what goes in each
- placeholder syntax that makes incomplete sections obvious (`[TBD: reason]`, `[STUB: blocking item]`)
- traceability fields (source document, section, date, author, status)
- status tags: `draft | review | approved | rtl_ready | not_rtl_ready`
- links/references to related documents in the set

The block spec template (`sfu_block_spec_template.md`) already has requirements from our project instructions — incorporate those, don't duplicate.

## Output format

Three outputs:

1. **Document set proposal** — numbered list with the fields above, in chat
2. **Upload shopping list** — ranked table of resources to upload, in chat  
3. **`sfu_document_templates.md`** — the consolidated template file, as artifact

After this, I'll upload the top resources you recommend, and we'll refine the templates with that input.