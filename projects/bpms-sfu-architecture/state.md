# bpms-sfu-architecture — canonical state

**Last updated:** 2026-04-21
**Mirrored to Claude Project:** BPMS SFU Architecture

## Scope

SFU (Satellite Frontend Unit) micro-architecture for BPMS 1.0 — AST
SpaceMobile ground gateway. Band-level processing unit. Deliverable
pipeline:

'system architecture docs → .md FPGA architecture and block specs
→ Claude Code → SystemVerilog RTL'

Dual objective: produce implementable, verifiable SFU hardware
micro-architecture, and build skill in deriving FPGA architecture and
RTL block specs from system-level docs.

## Current situation

Initial phase: establishing SFU functional boundary, signal chain, and
block-spec template. No block specs produced yet. `sfu_block_spec_template.md`
may need to be proposed before first real block spec.

## Load-bearing technical facts

**SFU ownership**

- Owns: OBG ingest, lane alignment, bins selection and routing,
  filter-bank synthesis and analysis, band gain, band Doppler, RF
  interfacing, timing, management, debug.
- Does NOT own: per-beam functions — beam gain, beam Doppler, beam
  delay compensation. Those belong to DCU.
- Any proposal that crosses DCU/SFU boundary must be flagged explicitly.

**RTL-ready criteria for a block spec**

A block spec is only RTL-ready when all of these are fixed:

- Interface signals: name, width, direction, clock, reset, handshake
  semantics (AXI4-Stream, valid/ready, PLIO, etc.)
- Clock and reset domains, CDC points, async-FIFO requirements
- Parameters: name, type, default, legal range
- Fixed-point formats in Q(I.F) or s(W,F) at every interface and
  significant internal node
- Algorithm / FSM in pseudocode or pointer to bit-exact reference model
- Latency, throughput, backpressure behaviour
- Register map or control interface when applicable
- Error / corner-case behaviour (overflow, saturation, reset-during-
  transfer, invalid input)
- Reference model pointer (Python / MATLAB) + bit-exactness requirement
- Verification hooks: counters, status bits, debug taps, injection points

Specs with TBD or STUB at any of these points are marked
`not_rtl_ready` with blocking items listed.

**Source of truth**

Uploaded project documents. Prefer more specific and more recent when
artifacts conflict, unless overridden by a higher-level source. Never
invent missing requirements. Always separate
*explicitly specified / inferred / proposed*. Cite document + section
for every spec claim.

## Architecture / design choices made

(none yet recorded — populate as decisions accumulate in `decisions/`)

## Open questions / decisions

- `sfu_block_spec_template.md` — exists yet, or propose structure first?
- Which SFU block is first to spec end-to-end?
- Fixed-point notation preference: Q(I.F) or s(W,F) as primary in this
  project? Template must pick one.

## Active references

- BPMS 1.0 System Architecture spec v1.3, S. Kulik (title + revision
  only — do not paste content)
- Further AST internal BPMS docs uploaded to the Claude Project KB

## File-naming conventions (for reusable outputs)

Snake case, Obsidian-ready Markdown:

- `sfu_architecture_review.md`
- `sfu_functional_boundary.md`
- `sfu_signal_chain.md`
- `sfu_block_spec_<block_name>.md`
- `sfu_block_spec_template.md`
- `sfu_open_questions.md`
- `sfu_assumptions_log.md`
- `sfu_verification_checklist.md`
- `sfu_adr_<short_topic>.md`

## Recent changes

- 2026-04-21: initial state seeded from Project instructions.
