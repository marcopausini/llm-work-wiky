# Task: revise FAD §2 (top-level block diagram) and §6 (module inventory) to spec-level

# granularity per ADR-0005

## Background

The first draft of FAD §2 and §6 was authored under the legacy MDS framework, where
the architect owned per-module documentation down to .sv-level decomposition.

ADR-0005 (spec/uarch split) changed this:

- Architect owns the **module spec (MS)** — external contract for a functional block
- RTL designer owns the **micro-architecture (uArch)** — internal decomposition,
  including how many .sv files implement the block
- Per-block granularity in the FAD is now **functional-block**, not .sv-level

The earlier draft therefore over-specifies: it lists sub-blocks that should not exist
at FAD scope, and conflates the architect's deliverable with the designer's.

## What to do

1. Load the existing FAD §2 and §6 drafts from project knowledge (point me at the
   filenames if they are not auto-discovered).
2. Read ADR-0005 and `arch/README.md` (spec/uarch terminology) and confirm the rule:
   **the FAD inventory is the canonical functional-block list; .sv-level decomposition
   is uArch territory and does not appear here.**
3. Walk §2 (top-level block diagram) and §6 (module inventory) row by row. For each
   item ask:
   - Is this a functional block with its own external contract (one MS)?
     → keep, at functional-block granularity
   - Is this a sub-block that exists only because the legacy draft anticipated an .sv
     boundary? → remove from §6 / collapse on the §2 diagram, note as designer-scope
     in a comment if useful
   - Is this a wrapper around a vendor IP? → keep as a single MS row with reuse type
     `wrap`
4. Produce the revised §2 and §6, plus a short delta table showing what changed and
   why (legacy row → new row → rationale citing ADR-0005 / arch/README.md / SFU-001 §).
5. Flag any boundary case where the right granularity is genuinely ambiguous and a
   short ADR amendment or follow-up note would help.

## Constraints

- **Do not invent functional blocks.** Every block in §6 must be traceable to
  SFU-001 §5–§7 (signal-chain blocks) or to a clearly-architectural responsibility
  (clocking, management, debug infrastructure).
- **Do not prescribe internal structure** in §2 or §6. No FSMs, no pipeline stages, no
  FIFO depths, no .sv filenames. Those belong to the designer's uArch (gated at G4b).
- **Keep §2 and §6 bidirectionally consistent** — every block in the §2 diagram has
  exactly one row in §6, and vice versa. The framework `make doc_check` rule
  (`04-execution-flow.md` §4) will enforce this; getting it right manually now avoids
  rework.
- **Cite every keep / drop / merge decision** in the delta table. Either to ADR-0005,
  to `arch/README.md`, or to a SFU-001 / ARCH-001 section.

## Deliverable

A single Markdown file containing:

1. Revised FAD §2 — top-level block diagram (text description, with placeholder for
   the actual diagram artefact path)
2. Revised FAD §6 — module inventory table (one row per functional block, with
   columns: block name, module-spec filename, reuse type, clock domains, owner,
   short purpose, traceability to SFU-001 §)
3. Delta table — old row → new row → rationale
4. Open questions / boundary cases (if any)

## Done criteria

- §6 row count matches the §2 block count
- Every row is functional-block-level, not .sv-level
- Every row has a module-spec filename matching `arch/modules/<snake_case>.md`
- No sub-block decomposition leaked from the legacy draft
- Delta table shows every change with a citation
- Output is ready to drop into the FAD draft, replacing the existing §2 and §6

Start by asking me to share or confirm the location of the existing FAD §2 and §6
draft, plus ADR-0005. Then propose your revision plan before writing the deliverable —
I want to see the row-by-row decisions before they land in the final document.
