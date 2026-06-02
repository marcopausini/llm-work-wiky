
---
---

# New chat prompt — Draft SFU FPGA top-level block diagram (FAD §2)

I am bootstrapping the FPGA architecture for the BPMS 1.0 Satellite Frontend Unit (SFU). The documentation framework and repo scaffold are settled. The next deliverable is the top-level block diagram for FAD §2.

**This is the first pass through the architecture documents.** No prior analysis exists. The FAD is an empty skeleton. Start from the source documents and derive everything fresh.

## Goal

Read the SFU architecture document, extract the signal chain, and draft the SFU FPGA top-level block diagram for FAD §2. The output is:

1. A module naming convention, agreed before any block is named.
2. The diagram(s) in Mermaid (preferred for diffability) ready to drop into FAD §2.
3. A preliminary RTL module list extracted from the diagram, ready to seed FAD §6 Module Inventory.

## Source documents

- **Primary:** `BPMS_1.0_SFU_Architecture_v1.6.docx` (BPMS-1.0-SFU-001) — the SFU functional architecture. This is the single source of truth for what the SFU does.
- **Secondary:** `BPMS_1.0_Architecture_Document_v2.4.docx` (BPMS-1.0-ARCH-001) — system-level context. Use only when SFU-001 explicitly references it or when system-level context is needed to understand SFU boundaries.

Both are uploaded to the project knowledge base. Read them; do not rely on assumptions about their content.

## Constraints

- Target: BittWare RFX-8440A (AMD Zynq UltraScale+ RFSoC XCZU43DR-2FFVE1156E, speed grade -2).
- Framework is 5-doc Markdown (FAD/MDS/ICD/ADR/RTM). Do not propose alternative document structures.
- Repo is `bpms-sfu-fpga-design/` with flat layout: `arch/ rtl/ model/ constraints/ prj/ scripts/ docs/`. Do not propose alternative repo layouts.
- Citation discipline: every factual claim cites SFU-001 §x or ARCH-001 §y. Markers are mandatory:
  - `[INFERRED from <§>]` — derived from a source doc but not literally stated there
  - `[ASSUMPTION: <text>, <expiry trigger>]` — implementation choice with no direct functional source
  - `[TBD: <reason>, <owner>]` — value not yet known
  - `[STUB: <blocking item>]` — section deliberately empty
- `state.md` is canonical; contradictions with chat or instructions are resolved in favour of `state.md`.

## Working principles for this diagram

1. **Read SFU-001 first.** List every functional block named or implied in §4–§7, §8, §11, §14–§15. Cite the section for each. Present this list before drawing anything.
2. **Separate DL and UL paths cleanly.** Two diagrams (one DL, one UL) are acceptable and likely preferable to one cluttered combined view. Shared infrastructure (RF port select, RFdc, mgmt, clocks) shown in both or in a third "shared" view — your call, justify it.
3. **Two-level decomposition.** A conceptual top-level diagram showing functional super-blocks, then per-super-block expansion to RTL modules if the full SFU does not fit clearly in a single readable view. Human readability is paramount at this phase.
4. **Color to group or separate functionalities.** Suggested groupings (refine after reading the docs): ingress/egress, DSP, control, debug. Use distinct arrow styles to separate dataplane from control plane (suggested: solid = data, dashed = control/CSR, dotted = clock/timing).
5. **Every block corresponds to an RTL module.** No conceptual blocks that don't map to a future MDS. Every named block traces to one of:
   - **(a)** an SFU-001 / ARCH-001 section that describes the function it implements (cite inline)
   - **(b)** `[INFERRED from <§>]` — a decomposition or aggregation derived from a cited functional description
   - **(c)** `[ASSUMPTION: <text>, <expiry trigger>]` — an implementation-driven block (CDC FIFO, register adapter, arbiter, clock buffer, etc.) with no direct functional source, justified inline
6. **The output seeds FAD §6.** Every block in the diagram becomes a row in the RTL module inventory.

## Module naming convention — define before naming any block

Propose a convention covering:
- **(a) Prefix scheme.** `sfu_<function>` vs unprefixed (since the repo is already SFU-scoped). Recommendation with rationale.
- **(b) DL/UL suffix policy** where the same function exists in both directions. Pick a style and apply consistently.
- **(c) Abbreviation rules.** When to expand, when to abbreviate. Suggest a hard rule (e.g. no abbreviations except a defined whitelist).
- **(d) Sub-block naming** for second-level decomposition. Naming scheme and Verilog instance naming implications.

Get explicit approval on the convention before applying it to any block.

## Process — ask, don't guess

Ask clarifying questions whenever the source documents leave a structural choice open or whenever multiple decompositions are defensible. Examples worth raising rather than guessing:

- Granularity of sub-band A/B — one block showing both vs one block per sub-band.
- Whether RFdc tile, DUC/DDC, and RF port select are one block or separated.
- Whether debug infrastructure (playback, capture, spur detection, beacon, RMS) appears as inline blocks or as tap/inject annotations.
- How the management plane fan-out is shown — single `mgmt_if` with dashed fan-out vs per-block CSR shown at top level.
- How clock/reset distribution is shown without crowding the diagram — likely a separate clock-tree view rather than wires on the data diagram.
- Whether the OBG Aurora ingress/egress is shown as four parallel lanes, one block representing all four, or one block per lane.

Surface options. Recommend one. Wait for approval.

## Suggested order of work

1. Read SFU-001 §4–§7 carefully. List every functional block named or implied. Cite the section for each. Present the list.
2. Read SFU-001 §8, §11, §14–§15 for debug, timing, M&C, and telemetry blocks. Add to the list.
3. Propose the module naming convention. Get approval.
4. Propose the DL conceptual top-level diagram (Mermaid). Get approval on block list and grouping/colour scheme before refining wires.
5. Propose the UL conceptual top-level diagram (Mermaid). Get approval.
6. Propose the shared-infrastructure view (mgmt, clocks, debug taps, RF port select if shared). Get approval.
7. For any super-block too dense to read at top level, draft the second-level expansion.
8. Emit the final diagram(s) and the seeded FAD §6 module inventory rows.

Open questions, options, and recommendations should be visible in the chat — not buried in the diagram.
