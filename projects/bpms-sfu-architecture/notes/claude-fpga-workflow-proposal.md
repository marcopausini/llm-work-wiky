# Claude FPGA Workflow — Proposal

**Author:** Marco Pausini
**Audience:** Shlomi Kulik (AVP System FPGA/ASIC Architect), Adi Michaeli (Design Team Leader), Gaston Rodriguez (Senior FPGA Engineer)
**Date:** YYYY-MM-DD
**Status:** Draft for discussion

---

## 1. Purpose

Propose a structured, Claude-assisted workflow for FPGA development at AST SpaceMobile, covering the full path from system architecture down to synthesizable RTL and unit-level testbenches. Short enough to be actionable this week; durable enough to scale to the team.

Anchored in the **V-model** (standard digital hardware lifecycle), trimmed to what a fast-moving, small team can realistically run. Where the classical V-model prescribes separate documents, we consolidate; where it prescribes process, we keep only what produces artefacts someone will actually read.

---

## 2. The two roles

FPGA work naturally splits into two roles, each with its own workflow, inputs, outputs, and Claude usage pattern.

| | FPGA Architect | FPGA RTL Design Engineer |
|---|---|---|
| **Input** | System architecture & requirements (BPMS SAD, SFU Arch Doc) | Module Design Spec (MDS) + ICDs |
| **Output** | FAD, MDSs, ICDs, ADRs, RTM | SystemVerilog/HLS modules + unit testbenches |
| **Primary Claude tool** | Claude Chat (design reasoning, doc authoring) | Claude Code (code generation, sim loop) |
| **Character of work** | Project-specific, low-volume, high-stakes | Repeatable, higher-volume, standardizable |
| **Typical Claude task** | "Is this decomposition consistent with the parent doc?" | "Generate RTL from this MDS. Generate cocotb TB. Iterate until it passes." |

**Contract between roles:** the **Module Design Spec (MDS)**. A well-written MDS is detailed enough that a junior RTL engineer — or Claude Code — can implement the module from it without further architectural input. This is the single most leveraged artefact in the workflow.

---

## 3. Workflow overview

```
System Architecture (Shlomi)              ──┐
                                            │  inputs
Requirements (system-level SyRS)          ──┘
                                            │
                            ┌───────────────┴───────────────┐
                            │     FPGA ARCHITECT TRACK      │
                            │                               │
                            │   FAD (device-level arch)     │
                            │   ICDs (shared protocols)     │
                            │   ADRs (decisions + why)      │
                            │   MDSs (per-module specs)  ───┼──┐
                            │   RTM (traceability)          │  │
                            └───────────────────────────────┘  │
                                                               │  handoff
                            ┌──────────────────────────────────┘
                            │
                            ▼
                            ┌───────────────────────────────┐
                            │   FPGA RTL DESIGN TRACK       │
                            │                               │
                            │   SystemVerilog / HLS         │
                            │   Unit testbenches (cocotb)   │
                            │   Lint / CDC / timing clean   │
                            └───────────────┬───────────────┘
                                            │
                                            ▼
                            System Integration & UVM (DV team)
```

### 3.1 V-model mapping

For reference — the V-model phases and where our artefacts land:

| V-model phase | Classical artefact | Our artefact |
|---|---|---|
| Concept / Mission | ConOps, MRD | (N/A at this level — upstream) |
| System Architecture | SAD, ICDs | BPMS SAD (exists), SFU Arch Doc (exists) |
| FPGA Requirements (HRD) | Separate FRS | **Inherited from SFU Arch Doc §18** — not duplicated |
| FPGA Architecture (ADD/HLD) | Separate ADD | **FAD** (new) |
| Module Design (DDD/LLD) | DDDs | **MDSs, one per module** (new) |
| Implementation | RTL | `code/` (new) |
| Unit Verification | Unit TB | Part of MDS sign-off |
| Integration Verification | UVM | Owned by DV team (exists) |

**What we deliberately skip:** separate HRD document (too much duplication with SFU Arch §18), separate VPlan (unit TB plan lives in each MDS; UVM plan owned by DV), formal PDR/CDR ceremony (we do lightweight internal reviews instead).

---

## 4. The five artefact types

Keep the total set small. Five types, covering both roles.

1. **FAD — FPGA Architecture Document** (1 per FPGA). Device-level architecture: top-level block diagram, clocking, CDC inventory, memory architecture, fixed-point policy, budgets, module inventory. Parent of all MDSs. Produced by the architect.

2. **MDS — Module Design Spec** (N, one per RTL module). Port list, parameters, register map, FSMs, pipeline stages, fixed-point formats, verification notes, edge cases. Detailed enough for a junior RTL engineer or Claude Code to implement directly. Produced by the architect, consumed by the RTL engineer.

3. **ICD — Interface Control Document** (~5, shared protocols). Single-source-of-truth for interfaces used across multiple modules: streaming bus format, register bus, OBG frame layout, register map. Produced by the architect, referenced by every MDS that uses the protocol.

4. **ADR — Architecture Decision Record** (N, as-needed). Short, dated, immutable-once-accepted capture of *why* a non-trivial decision was made. Alternatives considered, consequences accepted. Produced by the architect; durable artefact for future-you and future team members.

5. **RTM — Requirements Traceability Matrix** (1, live). Flat table: requirement → FAD section → module → unit test → UVM test. Updated continuously. This is the one artefact that cannot be allowed to rot; Claude maintains it mechanically.

Templates for all five are drafted and ready to apply to BPMS SFU.

---

## 5. Repository structure

### 5.1 Project repo — `bpms-sfu-fpga`

Contains BPMS-specific work. Two top-level decisions to make together:

- **Single repo vs. two repos** (TBD, to align with Gaston). Options:
  - *Single:* `bpms-sfu-fpga/` with `arch/` and `code/` subdirs. Claude Code sees both simultaneously. Simpler for traceability.
  - *Two:* `bpms-sfu-arch/` (docs + architect work) and `bpms-sfu-code/` (RTL + sim). Cleaner access control, but adds sync friction at the MDS boundary.

```
bpms-sfu-fpga/                 # TBD — may become two repos
├── arch/                      # FAD, MDSs, ICDs, ADRs, RTM
├── code/                      # rtl/, tb/, sim/, scripts/, vivado/
├── workflow/                  # pinned snapshot of claude-fpga-workflow (submodule)
├── CLAUDE.md                  # repo-level Claude Code guidance
└── README.md
```

### 5.2 Shared workflow repo — `claude-fpga-workflow`

Team-standard Claude commands, skills, templates, CLAUDE.md defaults. Versioned, consumed by every FPGA project repo via submodule or bootstrap script. Matures incrementally — v0.1 covers RTL-side commands, architect-side commands grow as patterns emerge.

### 5.3 Course repo — `claude-fpga-rtl-mentor`

Self-paced 5-module course teaching engineers how to use Claude Chat + Claude Code for FPGA RTL work. Main branch = curriculum; each student works on `<initials>/personal-session` branch. Outputs from personal branches feed the shared workflow repo via normal PRs.

---

## 6. Parallel tracks, one convergence point

Two work streams run in parallel:

**Track A — RTL workflow + toolset standardization.** Work through the mentor course, build personal toolset, converge an `module.md` (MDS) template that Claude Code consumes reliably. Led by Marco, with Gaston's review and contribution. Produces: `claude-fpga-workflow` v0.1.

**Track B — BPMS SFU architecture.** Read system architecture, draft FAD, draft first MDSs for BPMS SFU. Solo initially; reviewed by Shlomi. Produces: `bpms-sfu-fpga/arch/` populated.

**Convergence:** the MDS template. Track A discovers what format Claude Code implements well; Track B produces BPMS SFU content in that format. The MDS is the one artefact that must stabilize for the two tracks to join. **First MDS drafted only once the template is locked** — avoids churn from re-baselining docs drafted against a moving target.

### 6.1 Prior art — VeriFlow-CC

Track A builds on prior art rather than starting from scratch. We are evaluating **VeriFlow-CC** ([github.com/bjwanneng/veriflow-cc](https://github.com/bjwanneng/veriflow-cc)) — a Claude Code-driven single-module RTL pipeline with a staged architecture (requirement → spec → microarch → timing → coder → lint → sim → synth), a readiness-check gate before RTL generation, and a skill + sub-agent orchestration pattern. Scope is narrower than ours:

| | VeriFlow-CC | Our target |
|---|---|---|
| Scale | Single module (ALU, UART) | Multi-module FPGA (~20 modules) |
| Language | Verilog-2005 | SystemVerilog + cocotb |
| Toolchain | iverilog + yosys (FOSS) | Vivado + Versal + Vitis HLS |
| DSP | None | Fixed-point, filter banks, NCOs |
| Clocking | Single clock | Multi-clock with explicit CDC |
| Cross-module contracts | None | ICDs, RTM |

**Patterns we will adopt:** the staged pipeline with readiness gates, the two-artefact spec pattern (structured contract + behavioral prose), the 7-category Stage-1 clarity checklist, the skill + sub-agent architecture, structured logging with actual-string PASS/FAIL checks, the 3-retry budget with error-type-to-rollback-stage mapping.

**What we build ourselves:** SystemVerilog + cocotb code generation, Vivado/Versal toolchain hooks, fixed-point DSP awareness, multi-module/ICD/CDC support, integration with our MDS template and RTM.

This positions Track A as *adapt and extend* rather than *invent from scratch*, accelerating the timeline below.

---

## 7. Timeline and demonstrable milestones

| When | Track A (RTL workflow) | Track B (BPMS architecture) | Demonstrable to |
|---|---|---|---|
| Day 1–2 | Install VeriFlow-CC, run UART example end-to-end | Repo scaffolded; FAD skeleton + first ADRs committed | Shlomi, Gaston |
| Day 3–5 | Extract adoptable patterns; draft MDS readiness checklist | FAD §1–§6 filled (scope, top-level diagram, dataflow, clocking, module inventory) | Shlomi — "top-down chain is producing real documents" |
| Week 2 | MDS template v1.0 locked; skill + sub-agent adapted for SV/cocotb | First MDS drafted against locked template, handed end-to-end: MDS → Claude Code → SV RTL → passing cocotb unit TB | Manager — "the workflow works, end to end" |
| Week 3–4 | `claude-fpga-workflow` v0.1 stabilizing from real use on BPMS modules | 3–5 more MDSs through the chain | Team — "this is adoptable" |

**Why this sequencing:** FAD §1–§6 (device architecture, clocking, module inventory) does not depend on the MDS format — Track B stays productive while Track A figures out the MDS format against VeriFlow-CC's patterns. The first MDS is only drafted when the template is locked, so it doubles as the end-to-end demo.

The week-2 milestone is the key management-visible proof. Pick one small, self-contained module (e.g. RF port select, register bus adapter), walk it through the full chain, demonstrate the working result.

---

## 8. What the architect needs from the system architect

One thing that will materially accelerate Track B: the system architecture docs are in good shape already, but a few things help the FPGA architect consume them efficiently:

- **Stable requirement IDs** (SYS-xxx in SAD §22, SFU-xxx in SFU Arch §18). These IDs become the anchor of the RTM. Once baselined, they should not be renumbered — derived/new IDs are fine, but existing ones need to stick.
- **Explicit TBDs** flagged distinctly (the open-issues table already does this — perfect).
- **Source-of-truth awareness** — when a parameter appears in multiple sections, a single canonical location avoids drift. The SFU Arch Doc is already disciplined about this; worth maintaining as the docs evolve.

Nothing urgent; nothing to redo. Just flagging what we rely on as the BPMS 1.0 system architecture continues to mature in parallel with the FPGA architecture work.

---

## 9. Open decisions

Items to discuss and close before starting:

1. **One repo or two** for `bpms-sfu-fpga` — Marco + Gaston to decide.
2. **Workflow repo installation mechanism** (submodule vs. bootstrap script) — deferred to when first pulled into a project repo.
3. **Review cadence for FAD and MDSs** — suggest: lightweight review by Shlomi at FAD baseline, then review first 2 MDSs together to calibrate the template, then spot-check thereafter. Formal ceremony not required.
4. **Senior engineer time allocation for Gaston** on Track A — need rough commitment to size the v0.1 workflow effort honestly.

---

## 10. What this is not

To avoid false expectations:

- Not a replacement for DV. UVM ownership stays with the DV team. Unit testbenches per module (author-written) complement, not replace, system-level verification.
- Not an attempt to formalize AST's FPGA process to DO-254 / ECSS levels. Proportionate to team size and pace.
- Not a fixed syllabus. The module structure (FAD + MDSs + ICDs + ADRs + RTM) is stable; the Claude tooling around it will evolve as we learn.
- Not Claude-only. All artefacts are Markdown-in-Git, human-readable, reviewable via PR. Engineers who don't use Claude can still consume and contribute.

---

## 11. Next steps

1. Review this doc with Shlomi and Gaston.
2. Close the four open decisions in §9.
3. Start Track A (RTL workflow — Marco + Gaston) and Track B (BPMS SFU architecture — Marco, reviewed by Shlomi) in parallel this week.
4. Week-2 demo review.

---

## Appendix A — Terminology lock

To avoid confusion in future discussions:

| Term | Meaning |
|---|---|
| FAD | FPGA Architecture Document (device-level architecture) |
| MDS | Module Design Spec (per RTL module) |
| ICD | Interface Control Document (shared protocol) |
| ADR | Architecture Decision Record (non-trivial decisions) |
| RTM | Requirements Traceability Matrix |
| Architect track | Work producing FAD/MDS/ICD/ADR/RTM |
| RTL track | Work producing RTL + unit TBs from MDSs |
| Claude-assisted | Using Claude Chat and/or Claude Code as part of the workflow; all outputs are human-reviewable and git-tracked |
