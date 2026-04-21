# claude-fpga-rtl-mentor — canonical state

**Last updated:** 2026-04-21
**Mirrored to Claude Project:** Claude FPGA RTL Mentor

## Scope

Structured course teaching Marco how to leverage Claude Chat and Claude
Code inside FPGA RTL design and verification workflows. Not teaching FPGA
— teaching tool fluency for an already-expert FPGA engineer. Linear
module progression with pass-gated quizzes and concrete `scratch/`
artefacts per module.

## Current situation

Formally at M1 (Foundations). Informal preparation already done outside
the course: initial Claude Code exploration, draft CLAUDE.md for BPMS
v0.5, prompt sequence for architecture-map analysis, architecture-map
demo on BPMS v0.5 that surfaced potential CDC issues. None of that
counts as module completion.

## Load-bearing technical facts

- Linear module progression: M1 → M2 → M3 → M4 → M5. No skipping.
- Every module produces two outputs: an understanding milestone + a
  concrete artefact in `scratch/` on student's personal branch.
  Artefact non-negotiable.
- Pass threshold: quiz A+B ≥ 80%, Part C rubric ≥ 3/5. Below → repeat
  with different exercises.
- Implementation work (RTL, cocotb) happens in Claude Code in VS Code,
  not in the chat project. Mentor redirects if asked to write RTL.
- Validation target for artefacts: BPMS (Versal + AI Engine). Artefacts
  themselves must stay project-agnostic and free of company IP.
- Timeline: 3-day checkpoint = M1–M3 done + 1 command validated;
  5-day target = all 5 modules + 4 lab commands validated.

## Module → artefact map

| Module | Topic | Artefact |
|---|---|---|
| M1 | Foundations (Chat vs Code, agentic loop, plan/edit, context) | `scratch/` scaffolded on student branch |
| M2 | CLAUDE.md mastery | `scratch/templates/CLAUDE.md.template` |
| M3 | Commands vs Skills vs Agents vs MCP | `scratch/commands/arch-map.md` |
| M4 | Prompt engineering for RTL (Opus 4.7) | `scratch/commands/spec-from-rtl.md` + `rtl-from-spec.md` |
| M5 | Verification prompting + LLM limits for testbenches | `scratch/commands/cocotb-scaffold.md` |

## Open questions / decisions

- M1 start date — not yet fixed.
- Whether to request the three research documents (repo survey, use-case
  proposals, prompting best-practice) early in the course or after M3.

## Active references

- Claude Code docs
- Opus 4.7 prompting guidance (emerging, not yet well-documented)
- Reference repos (to be surveyed in research doc 01): VeriFlow-CC,
  ChipForge, RTL-Coder.

## Recent changes

- 2026-04-21: initial state seeded from Project instructions.
