---
id: 2026-04-22--claude-fpga-rtl-mentor--rtl-workflow-veriflow-cc-adoption
date: 2026-04-22
project: claude-fpga-rtl-mentor
type: decision
status: ongoing
topics: [workflow, veriflow-cc, skill, sub-agent, claude-code, cocotb, systemverilog, mds-contract]
source_chat: claude-opus-4.7
---

# RTL workflow bootstrap — VeriFlow-CC adoption and team-standard toolset

## Context
Decision session on how the FPGA RTL Design Engineer role will use Claude Code. Track A of a parallel workflow setup (Track B = BPMS SFU architecture). Identifies VeriFlow-CC (`github.com/bjwanneng/veriflow-cc`) as directly applicable prior art; defines the `claude-fpga-workflow` team-standard toolset repo separately from project repos. Companion checkpoint under `bpms-sfu-architecture` covers Track B.

## Key findings
- **VeriFlow-CC is real prior art.** Single-module RTL pipeline, 8 staged phases with readiness gate, skill + sub-agent orchestration, zero-pip-dependency. Narrower than AST target (single module, Verilog-2005, iverilog/yosys, no DSP, single clock, no ICDs) but the patterns are directly applicable. Adopt as Track A starting point — accelerates timeline and de-risks the skill+sub-agent architecture.
- **Two-role separation is load-bearing.** FPGA Architect (Claude Chat primary) produces MDSs; FPGA RTL Design Engineer (Claude Code primary) consumes MDSs. Different inputs, outputs, tool usage, prompting patterns, failure modes. Do not conflate.
- **MDS is the contract.** Single most leveraged artefact. RTL workflow is designed around consuming MDS cleanly; architect workflow around producing MDSs clean enough to consume.
- **Team-standard toolset lives in its own repo.** `claude-fpga-workflow` separate from mentor course repo (`claude-fpga-rtl-mentor`) and project repos (`bpms-sfu-fpga`). Consumed via submodule or bootstrap script.
- **Mentor repo stays curriculum-centric.** `main` = curriculum, `<initials>/personal-session` per student. Not the home for team workflow.

## Decisions
- **VeriFlow-CC patterns adopted into `claude-fpga-workflow`:**
  - Staged pipeline with readiness gates
  - Two-artefact spec pattern: structured contract + behavioural prose
  - 7-category Stage-1 clarity checklist (functional, constraints, design intent, algorithm/protocol, timing, domain knowledge, completeness meta-check)
  - Skill + sub-agent orchestration
  - Structured logging with literal PASS/FAIL string checks
  - 3-retry budget with error-type → rollback-stage mapping
  - Multi-file input structure: `requirement.md`, `constraints.md`, `design_intent.md`, `context/*.md`
- **AST builds on top (not covered by VeriFlow-CC):**
  - SV + cocotb codegen (VeriFlow-CC is Verilog-2005 + iverilog/yosys)
  - Vivado / Versal / Vitis HLS toolchain hooks
  - Fixed-point DSP awareness (Q-formats, growth, saturation)
  - Multi-module, ICD, CDC support
  - Integration with AST MDS template + RTM
- **Repo structure:**
  - `claude-fpga-rtl-mentor` — existing course; `main` = curriculum, `<initials>/personal-session` per student.
  - `claude-fpga-workflow` — new, team-standard skills/commands/CLAUDE.md/templates.
- **Chat project rename:** `[Claude-FPGA] Mentor` → `[Claude-FPGA] RTL Design Mentor`.
- **Role titles:** FPGA RTL Design Engineer / RTL Design Engineer / Digital Design Engineer (implementer side). Architect side: FPGA Architect / Design Lead / Principal/Staff. "FPGA Architect" or "FPGA Design Lead" accurate for Marco's current work.
- **File format:** Markdown-in-Git, Obsidian-friendly, human-reviewable PRs.

## Open questions
- Workflow installation mechanism — submodule vs. bootstrap script. Deferred to v0.2.
- Version pinning strategy for `claude-fpga-workflow` consumed by project repos. Deferred to v0.2.
- Architect-side commands (RTM auto-regen, cross-reference linting, template conformance) — build only when pattern emerges from repeated manual pain. Not v0.1.

## Action items
- [ ] Rename `[Claude-FPGA] Mentor` Claude Chat project to `[Claude-FPGA] RTL Design Mentor`.
- [ ] **Day 1–2:** install VeriFlow-CC locally, run UART example end-to-end, capture observations.
- [ ] **Day 3–5:** extract reusable patterns from VeriFlow-CC; draft MDS readiness checklist (based on 7-cat Stage-1 clarity checklist, adapted for SV + cocotb + fixed-point + ICDs).
- [ ] Create `claude-fpga-workflow` repo. Scaffold: `skills/`, `commands/`, `CLAUDE.md`, `templates/` (MDS readiness checklist, unit-TB skeleton, CLAUDE.md seed for project repos).
- [ ] **Week 2:** lock MDS template v1.0. Adapt VeriFlow-CC skill + sub-agent for SV / cocotb / fixed-point.
- [ ] **Week 2 end goal (may slip to week 3):** first MDS → Claude Code → SV RTL → passing cocotb TB, end-to-end demo.
- [ ] **Week 3–4:** `claude-fpga-workflow` v0.1 stabilises; 3–5 more MDSs flow through.

## Re-seed block

GOAL: stand up `claude-fpga-workflow` v0.1 (skills, commands, CLAUDE.md, MDS readiness checklist, unit-TB skeleton) such that the FPGA RTL Design Engineer role can consume an MDS + ICDs and produce SV + passing cocotb unit TB via Claude Code, with 3-retry rollback behaviour.
CONSTRAINTS:
- Adopt VeriFlow-CC (`github.com/bjwanneng/veriflow-cc`) patterns; do not reinvent. Narrow scope of upstream (single module, Verilog-2005, iverilog/yosys, no DSP, single clock, no ICDs) — AST must add SV+cocotb, Vivado/Versal/Vitis HLS, fixed-point DSP, multi-module/ICD/CDC, MDS+RTM integration.
- `claude-fpga-workflow` is a separate repo from both `claude-fpga-rtl-mentor` (curriculum) and `bpms-sfu-fpga` (project). Consumed via submodule or bootstrap — mechanism deferred to v0.2.
- Architect-side commands (RTM regen, linting, conformance) are out-of-scope for v0.1 — build only when recurring manual pain appears.
- Markdown-in-Git, Obsidian-friendly, human-reviewable PRs.
- Solo on both tracks; realistic week-2 end-to-end demo may slip to week 3.
PRIOR CONCLUSIONS:
- VeriFlow-CC patterns adopted verbatim where applicable: staged pipeline + readiness gates, 7-cat Stage-1 checklist, skill+sub-agent, structured PASS/FAIL logging, 3-retry budget with error→rollback map, multi-file input (`requirement.md`, `constraints.md`, `design_intent.md`, `context/*.md`).
- Two roles separated: FPGA Architect (Chat, produces FAD/MDS/ICD/ADR/RTM) vs. FPGA RTL Design Engineer (Code, produces SV/HLS + unit TB). MDS is the contract.
- Mentor chat project renamed to `[Claude-FPGA] RTL Design Mentor`.
- Three repos: `claude-fpga-rtl-mentor` (course, unchanged), `claude-fpga-workflow` (new team toolset), `bpms-sfu-fpga` (new project — Track B).
CURRENT QUESTION: after VeriFlow-CC UART example runs end-to-end locally, which single pattern is extracted and SV+cocotb-adapted first — the readiness gate, the skill+sub-agent orchestration, or the 3-retry error→rollback map — to deliver the v0.1 vertical slice fastest?

→ Companion checkpoint (same date, BPMS scope): `2026-04-22--bpms-sfu-architecture--fpga-workflow-and-repo-setup`.
