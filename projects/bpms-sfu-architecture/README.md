# bpms-sfu-architecture

BPMS system architecture work, SFU-focused. Ground-based Beam Processing
and Management System bridging LTE/5G eNodeBs to LEO satellites over
QV-band.

## Files

- `state.md` — canonical current state. Uploaded to the Claude Project KB.
  Tracks: current architecture phase, design choices made, open questions,
  references to AST internal specs (by title and revision only).
- `notes.md` — working scratchpad.
- `checkpoints/` — per-chat artifacts from the `checkpoint` skill.
- `decisions/` — ADRs for architecture choices (clocking, partitioning,
  interfaces, device selection, …).

## IP handling

- Reference AST internal specs by title + revision only
  (e.g. "BPMS 1.0 v1.3, S. Kulik").
- Never paste spec sections or diagrams verbatim.
- When a checkpoint's reasoning depends on spec content, compress your own
  conclusions and cite the section; do not quote it.
- No partner or customer identifiers.

## Claude Project mapping

Project name: **bpms-sfu-architecture**. KB contents: `state.md` from this
folder, plus sanitised reference extracts if needed. Project instructions
note: *"Canonical state is the uploaded state.md. Do not solicit or
reproduce verbatim AST internal spec content."*
