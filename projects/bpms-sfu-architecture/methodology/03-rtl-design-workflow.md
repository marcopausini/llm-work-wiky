---
doc_id: BPMS-SFU-METH-003
doc_type: methodology
project: bpms-sfu-fpga-design
status: draft
version: 0.1
date: 2026-04-26
author: TBD
---

# FPGA RTL Design Workflow

[STUB: Content to be authored when the first module reaches `rtl_ready` and RTL
implementation begins. This document will define the RTL designer's workflow from
MDS consumption through implementation evidence capture.]

Parent document: [fpga-project-methodology.md](fpga-project-methodology.md).

---

## Planned scope

- RTL designer role definition and responsibilities
- Input consumption: MDS, ICDs, FAD conventions, reference models
- RTL coding conventions and style guide
- Unit testbench authoring process
- Reference model correlation workflow
- Lint, simulation, and OOC synthesis workflow
- Implementation evidence capture (MDS §12.3)
- Integration gate criteria
- LLM-assisted RTL generation (Claude Code) guidelines
- Handoff to integration and DV

---

## References

- [fpga-project-methodology.md](fpga-project-methodology.md) — top-level methodology
- [architect-workflow.md](architect-workflow.md) — architect workflow and handoff spec
- [execution-flow.md](execution-flow.md) — tool commands, CI gates, report requirements
- [signoff-criteria.md](signoff-criteria.md) — sign-off criteria per level
