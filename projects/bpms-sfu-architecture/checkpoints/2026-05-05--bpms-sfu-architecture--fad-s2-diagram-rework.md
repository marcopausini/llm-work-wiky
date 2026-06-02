---
id: 2026-05-05--bpms-sfu-architecture--fad-s2-diagram-rework
date: 2026-05-05
project: bpms-sfu-architecture
type: decision
status: ongoing
topics: [fad, datapath-diagram, infrastructure-diagram, drawio, sub-band-architecture, rf-port-sel]
source_chat: claude-opus-4.6
---

# FAD §2 diagram rework — drawio migration + infrastructure layout fix

## Context

Continuing FAD revision after the §1/§2/§6 restructuring (checkpoint 2026-05-01). Marco created `.drawio` diagrams for the top-level datapath and sub-band processing, replacing the earlier Mermaid-based diagrams. Also created a `.drawio` infrastructure diagram and a standalone narrative description for each view. This session reviewed all three diagrams for correctness, produced the updated FAD §2 and §3, and fixed layout issues in the infrastructure diagram.

## Key findings

- **Top-level datapath diagram** (`top_datapath.drawio`): correct. DL and UL in single bidirectional view, shared blocks (`obg_link`, `rfdc_wrap`, `rf_port_sel`) shown once. All block ordering matches SFU-001 §5–§7.
- **Sub-band processing diagram** (`subband_processing.drawio`): correct. DL/UL asymmetry accurately reflects SFU-001 §6 vs §7 ordering. Quantitative annotations (4×5,760 bins/lane, 12,288-bin frame, ≤9,600 occupied bins, 2×1,024 Msps) verified against SFU-001 §6.4, §12.1, §12.2. RMS tap points match SFU-001 §14.2.
- **Infrastructure diagram** (`infrastructure.drawio`): topology correct (all blocks, connections, line styles verified against ARCH-001 §15.2–§15.3, SFU-001 §6.6, §7.2, §11.1–§11.4, §14.2, §15.4, §17.1–§17.2). However, **seven layout issues** identified and fixed:
  - 3 region labels ("Management plane", "Timing / clock plane", "Datapath endpoints") hidden behind blocks — fixed by moving labels to top-left of background rects.
  - 3 arrow-crosses-block problems: `reg_bus→event_log` crossed `nv_cfg` and `band_doppler`; `utc_apply→utc_endpts` crossed `tle_compute`; `pps→event_log` crossed `band_doppler` — fixed by rerouting through clear corridors.
  - 1 boundary overflow: `utc_apply_engine` and `tle_compute` exceeded management plane background rect — fixed by widening rect.
- **§3 Dataflow collapsed**: narrative absorbed into §2 directly beneath each figure. §3 retained as redirect to preserve downstream numbering.
- **Mermaid fully retired** from §2.1 and §2.2. Retained only for §2.3 infrastructure (pending future `.drawio` migration, tracked as FAD-OQ-05). After this session, §2.3 also has a `.drawio` — Mermaid can be fully retired if desired.

## Decisions

- FAD §2 reorganised into §2.1 (top-level datapath), §2.2 (sub-band processing detail), §2.3 (infrastructure). Narrative folded directly beneath each diagram figure.
- `.drawio` files are the single editable source of truth for all three diagram views. SVGs are rendered exports.
- §3 skeleton preserved as a numbering anchor (empty redirect to §2).
- Infrastructure `.drawio` layout fixes applied programmatically (Python/XML); all seven issues verified clean.
- FAD-DEC-02 updated to reflect three diagram views.
- FAD-DEC-08 clarified: `rf_port_sel` is a digital PL-fabric mux between band-level DSP and `rfdc_wrap`.

## Open questions

- FAD-OQ-05: now that all three views have `.drawio` sources, decide whether to retire the §2.3 Mermaid entirely (replace with `.drawio` SVG embed). Low priority.
- Infrastructure `.drawio` needs re-export to `.svg` after layout fixes — Marco must open in draw.io and re-export.

## Action items

- [ ] Open `infrastructure.drawio` (fixed version) in draw.io, visually confirm layout, re-export as `infrastructure.svg`
- [ ] Replace FAD.md in repo with FAD v0.3 content (delivered this session)
- [ ] Replace §2.3 in FAD v0.3 with the updated infrastructure section (delivered this session as `FAD_section_2_3.md`)
- [ ] Copy `.drawio` + `.svg` files to `arch/fad/diagrams/`
- [ ] Delete legacy Mermaid diagram files (`datapath.md`, `dl-datapath.md`, `ul-datapath.md`)
- [ ] Commit FAD v0.3 to repo
- [ ] Begin FAD §4 (Clocking and Reset Architecture) — next major authoring task
- [ ] Begin core ICDs (streaming_bus, register_bus, obg_frame) — prerequisite for MS authoring

## Artefacts produced

- `FAD_v0.3.md` — full updated FAD with §1 (unchanged from v0.2), §2 reworked (three diagram views with inline narrative), §3 collapsed, §6 (unchanged from v0.2), §13 open questions updated, §15 change log updated.
  → target: `arch/fad/FAD.md` (replaces current)
- `FAD_section_2_3.md` — standalone updated §2.3 Infrastructure / Shared View with `.drawio` reference and tightened narrative.
  → target: merge into `arch/fad/FAD.md` §2.3 (replaces the Mermaid version in FAD_v0.3.md)
- `infrastructure.drawio` (fixed) — layout-corrected infrastructure diagram with all seven issues resolved.
  → target: `arch/fad/diagrams/infrastructure.drawio` (replaces original)

## Re-seed block

GOAL: Continue FAD authoring — next sections are §4 (Clocking and Reset Architecture) and core ICDs.
CONSTRAINTS:
- Target device: BittWare RFX-8440A (AMD XCZU43DR-2FFVE1156E, speed grade -2)
- ADR-0005 accepted: MS = architect-owned external contract; uArch = designer-owned
- Gastón's template confirmation (Task 8) remains critical-path blocker for MS authoring
- `.drawio` is the single diagram source of truth; SVGs are rendered exports
PRIOR CONCLUSIONS:
- FAD v0.3 complete through §2 (three diagram views with narrative) and §6 (23 MS files, ~28 RTL instances)
- All three diagrams verified correct against ARCH-001 v2.4 and SFU-001 v1.6
- Infrastructure layout issues (7) identified and fixed programmatically
- §3 collapsed; narrative absorbed into §2; downstream numbering preserved
- CDC methodology finalised: FAD §4.5 owns policy; Vivado report_cdc handles per-flop enumeration
- Terminology cleanup complete (MDS→MS, rtl_ready→design_ready)
CURRENT QUESTION: Author FAD §4 (clock inventory, clock topology, primary/secondary modes, reset topology, CDC inventory) and begin core ICDs (streaming_bus, register_bus, obg_frame) to unblock MS authoring.
