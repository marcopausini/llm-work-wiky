---
id: 2026-04-22--bpms-sfu-architecture--fpga-workflow-and-repo-setup
date: 2026-04-22
project: bpms-sfu-architecture
type: decision
status: ongoing
topics: [workflow, v-model, repo-structure, fad, mds, icd, adr, rtm, claude-code, veriflow-cc]
source_chat: claude-opus-4.7
---

# BPMS SFU FPGA workflow, doc set, and repo setup — decisions locked pre-meeting

## Context
Pre-meeting consolidation with Shlomi / Adi / Gaston on how FPGA architecture and RTL work will flow for BPMS 1.0 SFU. Tailors V-model for fast-paced small team; sets doc set, repo structure, and Claude tooling boundary against BPMS 1.0 SAD v2.4 and SFU Arch Doc v1.6. Companion checkpoint under `claude-fpga-rtl-mentor` covers the RTL-mentor track.

## Key findings
- SFU Arch Doc v1.6 §18 and BPMS SAD v2.4 §22 requirements are usable as-is as HRD substitute. Parent docs do not need rewriting; only asks are stable IDs, explicit TBDs, single source of truth — already practised.
- MDS is the contract artefact between FPGA Architect and FPGA RTL Design Engineer. Single most leveraged doc in the chain. Must be detailed enough for junior RTL engineer or Claude Code to implement without architectural clarification.
- Document set is five types total: FAD, MDS, ICD, ADR, RTM. No separate HRD (inherit SFU Arch §18). No separate VPlan (unit TB in MDS, UVM owned by DV). No formal PDR/CDR.
- Two distinct roles: **FPGA Architect** produces FAD/MDS/ICD/ADR/RTM from system requirements; **FPGA RTL Design Engineer** consumes MDS + ICDs, produces SV/HLS + unit TBs. Different Claude tool usage (Chat vs. Code).
- Track A (RTL workflow standardisation) and Track B (BPMS architecture) run in parallel. Convergence point: MDS template. First BPMS MDS drafted only after template locks, to avoid re-baselining churn.
- Solo-on-both-tracks realism: days 1–5 = 27–38 h focused work. Week-2 end-to-end demo slips to week 3 if Gaston not committed — state honestly in meeting if asked.

## Decisions
- **Doc set (5 types):** FAD (per FPGA, device-level), MDS (per RTL module), ICD (shared protocol), ADR (decision record), RTM (live, per project).
- **V-model tailoring:** collapse HRD → SFU Arch §18; collapse VPlan → MDS §11 + DV UVM plan; drop PDR/CDR ceremony; skip ConOps/MRD.
- **Repos (three):**
  - `claude-fpga-rtl-mentor` — existing course; `main` = curriculum, `<initials>/personal-session` per student.
  - `claude-fpga-workflow` — new team-standard toolset (skills, commands, CLAUDE.md, templates). Consumed via submodule or bootstrap.
  - `bpms-sfu-fpga` — new BPMS project repo; `arch/` + `code/` + pinned `workflow/`. Single-vs-two split deferred to Gaston.
- **AST builds itself on top:** SV + cocotb codegen; Vivado / Versal / Vitis HLS hooks; fixed-point DSP awareness (Q-formats, growth, saturation); multi-module / ICD / CDC support; AST MDS template + RTM integration.
- **File format:** Markdown-in-Git, Obsidian-friendly, human-reviewable PRs.
- **Terminology locked:** FAD, MDS, ICD, ADR, RTM; Architect track vs. RTL track.

## Proposal
- **VeriFlow-CC patterns:** staged pipeline with readiness gates; two-artefact spec pattern (structured contract + behavioural prose); 7-category Stage-1 clarity checklist; skill + sub-agent orchestration; structured logging with literal PASS/FAIL; 3-retry budget with error→rollback-stage mapping; multi-file input structure (`requirement.md`, `constraints.md`, `design_intent.md`, `context/*.md`).


## Open questions
- `bpms-sfu-fpga` one repo (`arch/` + `code/`) vs. two (`bpms-sfu-arch` + `bpms-sfu-code`). Recommendation: one, for Claude Code multi-context access + MDS-boundary sync. Decide with Gaston.
- Workflow installation: submodule vs. bootstrap script. Deferred to v0.2.
- Review cadence for FAD and MDSs. Proposed: Shlomi reviews FAD baseline + first 2 MDSs together to calibrate, spot-checks after.
- Gaston's committed time on Track A — gates whether week-2 end-to-end demo is realistic.

## Action items
- [ ] Send meeting invite to Shlomi, Adi, Gaston with `fpga_claude_workflow.md` attached.
- [ ] **After meeting:** close the four open questions above.
- [ ] Create `bpms-sfu-fpga` repo (single or split per decision).
- [ ] Commit FAD / MDS / ICD / ADR / RTM templates from this chat to `bpms-sfu-fpga/docs/`.
- [ ] Draft ADR-0001..0005: clocking, streaming bus, register bus, fixed-point policy, refmodel language.
- [ ] Fill FAD §1, §2, §4, §6 from SFU Arch Doc v1.6 content.
- [ ] Week-2 demo: pick small self-contained module (RF port select or register bus adapter) for end-to-end walkthrough.
- [ ] Draft ICD templates for `streaming_bus`, `register_bus`, `obg_frame`, `obg_aurora`, `mgmt_regmap`.

## Artefacts produced
- `bpms-sfu-fpga/docs/README.md` — doc-set overview, authoring order, Claude workflow notes, folder layout.
- `bpms-sfu-fpga/docs/fad/FAD.md` — FPGA Architecture Document skeleton, 15 sections, pre-populated module inventory derived from SFU Arch.
- `bpms-sfu-fpga/docs/modules/_template.md` — MDS template, 14 sections, port/register/parameter tables, FSM Mermaid, fixed-point Q-format table, verification handoff (unit-TB vs. UVM), Claude Code implementation hints.
- `bpms-sfu-fpga/docs/icd/_template.md` — ICD template: signal list, payload/sideband, timing, rules, error behaviour, version policy.
- `bpms-sfu-fpga/docs/adr/_template.md` — ADR template: context, decision, alternatives, consequences, references.
- `bpms-sfu-fpga/docs/rtm.md` — RTM pre-seeded with every SFU-SIG/RF/CLK/LAT/MGT/DBG req from SFU Arch §18; columns FAD §, module, unit TB, UVM, status, evidence.
- `bpms-sfu-fpga/docs/fpga_claude_workflow.md` — management-facing proposal (234 lines): roles, workflow + V-model mapping, five artefact types, repo structure, parallel tracks, VeriFlow-CC adoption table, timeline, asks, open decisions, scope-honesty section.
- `bpms-sfu-fpga/docs/checkpoints/fpga_claude_workflow_decisions_checkpoint.md` — project-scoped checkpoint note capturing decisions, artefacts, re-seed block.

## Re-seed block

GOAL: stand up `bpms-sfu-fpga` repo with FAD/MDS/ICD/ADR/RTM templates filled, FAD §1/§2/§4/§6 populated from SFU Arch v1.6, and first MDS (small self-contained SFU block) flowing through Claude Code to SV + passing cocotb TB end-to-end.
CONSTRAINTS:
- Document set fixed at 5 types: FAD, MDS, ICD, ADR, RTM. No HRD, no separate VPlan, no PDR/CDR.
- SFU Arch Doc v1.6 §18 and BPMS SAD v2.4 §22 are the HRD substitute — do not rewrite parent specs.
- DCU/SFU ownership boundary: SFU owns OBG ingest, lane alignment, bins select, filter-bank synth/analysis, band gain, band Doppler, RF, timing, mgmt, debug. Beam-level (gain/Doppler/delay) is DCU — flag any crossover.
- Markdown-in-Git, Obsidian-friendly, human-reviewable PRs.
- MDS must be RTL-ready (interfaces, clocks/reset, Q-formats at every interface and node, FSM/algo pseudocode, latency/throughput/backpressure, regmap, error behaviour, refmodel pointer, verification hooks) before Claude Code consumes it.
- Solo on both tracks; realistic week-2 demo may slip to week 3.
PRIOR CONCLUSIONS:
- VeriFlow-CC patterns adopted (staged pipeline + readiness gates, 7-cat Stage-1 checklist, skill + sub-agent, 3-retry rollback, multi-file input); AST adds SV+cocotb, Vivado/Versal/Vitis HLS, fixed-point DSP, multi-module/ICD/CDC, MDS+RTM integration.
- Three repos: `claude-fpga-rtl-mentor` (course), `claude-fpga-workflow` (team toolset), `bpms-sfu-fpga` (project).
- First MDS drafted only after template lock — avoid re-baselining churn.
- Claude Chat project renamed to `[BPMS-1.0][SFU] FPGA Architect`.
CURRENT QUESTION: after the Shlomi/Adi/Gaston meeting closes the four open questions (single-vs-split BPMS repo, workflow install mechanism, review cadence, Gaston's Track A commitment), which SFU block is drafted first as end-to-end MDS→SV→cocotb demo — RF port select, or register bus adapter?

→ Companion checkpoint (same date, RTL mentor scope): `2026-04-22--claude-fpga-rtl-mentor--rtl-workflow-veriflow-cc-adoption`.
