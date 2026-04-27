---
doc_id: BPMS-SFU-METH-004
doc_type: methodology
project: bpms-sfu-fpga-design
status: draft
version: 0.2
date: 2026-04-27
author: Marco Pausini
---

# FPGA Execution Flow

This document defines the executable engineering flow for the BPMS 1.0 SFU FPGA project: tool commands, build entrypoints, gate definitions, IP integration policy, waiver policy, and report archival.

It is the *how-to-run-the-build* counterpart to [05-signoff-criteria.md](05-signoff-criteria.md) (which defines *what passing means*). Every flow gate in this document terminates with a cross-reference to the matching sign-off section.

Parent document: [01-fpga-project-methodology.md](01-fpga-project-methodology.md).

---

## 1. Purpose and scope

In scope:

- Tool baseline (Vivado, simulator, lint tool, target device)
- Build entrypoints (`make` targets and what each one runs)
- Per-stage gate commands and required reports
- Constraints and CDC methodology cross-references
- IP integration policy
- Waiver policy and storage
- Report archival convention
- CI regression levels
- Claude-assisted usage policy

Out of scope (covered elsewhere):

- Sign-off pass/fail criteria → [05-signoff-criteria.md](05-signoff-criteria.md)
- XDC file structure and ownership → [../../constraints/README.md](../../constraints/README.md)
- Architecture invariants → [../../arch/fad/_template.md](../../arch/fad/_template.md) §1.5
- CDC inventory (the design's actual CDCs) → FAD §4.5

---

## 2. Tool baseline

The tool baseline is pinned per project and updated only via ADR. A change to any row below requires a new regression baseline.

| Item | Value | Notes |
|---|---|---|
| Target device | `XCZU43DR-2FFVE1156E` | AMD Zynq UltraScale+ RFSoC, BittWare RFX-8440A |
| Synthesis / implementation | `[TBD: Vivado version, Architect]` | Pinned at first top-level synthesis run |
| Simulator | `[TBD: Questa / Xcelium / xsim, RTL Designer]` | Required for unit TB and OOC sim |
| Lint tool | `[TBD: Spyglass / Verilator-lint / SVTB-lint, RTL Designer]` | Used at every commit |
| CDC tool | `Vivado report_cdc` (primary), `[TBD: third-party CDC tool]` | Both used at sign-off |
| Reference model language | Python | Per ADR (numerical refmodels in `model/`) |
| Unit TB framework | `[TBD: cocotb / SystemVerilog UVM-light, RTL Designer]` | Per ADR |
| Build orchestrator | `make` (primary), Tcl scripts under `prj/` for Vivado | See §3 |

A change to the tool baseline that affects timing closure, simulation outcome, or CDC analysis is recorded as an ADR and triggers full regression.

---

## 3. Build entrypoints

All flow stages run from the repo root via a single `Makefile` (under `scripts/`) that wraps Vivado, the simulator, and the lint tool. Targets are idempotent: a clean checkout + this baseline → identical reports.

| Target | Stage | Inputs | Outputs |
|---|---|---|---|
| `make doc_check` | Documentation gate (§4) | `arch/`, MDS frontmatter | `reports/<commit>/doc_check.log` |
| `make lint MOD=<m>` | RTL lint (§5) | `rtl/<m>/`, lint config | `reports/<commit>/<m>/lint.log` |
| `make sim_unit MOD=<m>` | Unit sim (§6) | `tb/<m>/`, `model/<m>/` | `reports/<commit>/<m>/unit_sim.xml`, `coverage.rpt` |
| `make synth_ooc MOD=<m>` | OOC synthesis (§7) | `rtl/<m>/`, `constraints/ooc/<m>.xdc` | `reports/<commit>/<m>/synth_ooc.rpt`, `timing_summary.rpt` |
| `make cdc MOD=<m>` | CDC check (§11) | `rtl/<m>/` post-elab | `reports/<commit>/<m>/cdc.rpt` |
| `make synth_top` | Top synthesis (§8) | full RTL hierarchy + IP `.xci` | `reports/<commit>/top/synth.rpt`, `utilization.rpt` |
| `make impl` | Implementation (§9) | post-synth checkpoint | `reports/<commit>/top/impl_*.rpt` |
| `make timing` | Top timing | post-impl checkpoint | `reports/<commit>/top/timing_summary.rpt`, `check_timing.rpt` |
| `make reports` | Aggregate | all of the above | `reports/<commit>/INDEX.md` |
| `make clean` | Cleanup | — | removes local generated artefacts (does not touch committed files) |

Per-target flag conventions:

- `MOD=<module_name>` selects a single module (required for per-module targets).
- `IP=<ip_name>` selects a vendor IP for IP-only regeneration (`make ip_gen IP=rfdc`).
- `LEVEL=L0|L1|L2|L3|L4` selects a CI regression level (§14).

---

## 4. Documentation check gate

**Purpose:** prevent broken cross-references and unmarked gaps from reaching the integration gate.

**Command:** `make doc_check`

**Checks (script enforced when authored — see §16):**

1. Every block in FAD §2 (top-level diagram) appears in FAD §6 (module inventory) and has a matching `arch/modules/<module>.md`.
2. Every CDC named in any MDS §6.3 also appears in FAD §4.5.
3. Every MDS with `status: rtl_ready` has empty `rtl_ready_blocking:` and §11.7 fully checked.
4. Every MDS with `status: rtl_ready` referenced by an integrated module exists.
5. Every placeholder marker (`[TBD]`, `[STUB]`, `[ASSUMPTION]`, `[INFERRED]`) has the required fields.
6. Every factual claim has a citation OR a `[INFERRED]` / `[ASSUMPTION]` marker.

**Reports:** `reports/<commit>/doc_check.log` — one line per failure, exit non-zero on any failure.

**Sign-off criteria:** see [05-signoff-criteria.md](05-signoff-criteria.md) §3 (documentation sign-off).

---

## 5. RTL lint gate

**Purpose:** catch coding-rule violations, latches, unused signals, multi-driven nets, and protocol violations before simulation.

**Command:** `make lint MOD=<m>`

**What it runs:**

1. Lint tool with project rule set (lives under `scripts/lint/`).
2. SVA / property checks on protocol contracts from the MDS.
3. Naming-convention check.

**Reports:** `reports/<commit>/<m>/lint.log` — JSON or text, must include rule code, severity, file:line, description.

**Waivers:** entries in `waivers/lint/<m>.waiver` with required fields per [../../waivers/README.md](../../waivers/README.md).

**Sign-off criteria:** see [05-signoff-criteria.md](05-signoff-criteria.md) §4 (RTL lint sign-off).

---

## 6. Unit simulation gate

**Purpose:** prove RTL matches its specification at the module boundary, with bit-exact correlation against the reference model where the MDS bit-exactness policy requires it.

**Command:** `make sim_unit MOD=<m>`

**What it runs:**

1. Compile RTL + tb + dependencies (including vendor sim models).
2. Run all directed and constrained-random tests in `tb/<m>/`.
3. For `bit_exact` modules: read stimuli + golden vectors from `model/<m>/`, compare RTL outputs sample-by-sample.
4. Collect functional, FSM, and code coverage.

**Bit-exactness handling:**

| Policy (MDS §3.3) | Acceptance criterion |
|---|---|
| `bit_exact` | Zero LSB difference at every sample on the measurement boundary stated in MDS §7.3. |
| `ulp_bounded (bound=N)` | Every sample within ±N ULP at the measurement boundary. |
| `not_applicable` (vendor IP opaque) | Boundary metric (e.g. spectral mask, EVM) per MDS §7.3. |

**Reports:**

- `reports/<commit>/<m>/unit_sim.xml` — JUnit-style test results
- `reports/<commit>/<m>/coverage.rpt` — line / toggle / branch / functional / FSM coverage

**Sign-off criteria:** see [05-signoff-criteria.md](05-signoff-criteria.md) §5 (unit simulation sign-off).

---

## 7. Out-of-context synthesis gate

**Purpose:** prove a single module synthesises cleanly against its OOC constraints (its own clock period, virtual I/O delays) and meets per-module Fmax with margin — independent of top-level integration.

**Command:** `make synth_ooc MOD=<m>`

**What it runs:**

1. Vivado `synth_design -mode out_of_context` against `rtl/<m>/` + `constraints/ooc/<m>.xdc`.
2. `report_timing_summary`, `report_utilization`.
3. Resource compared against MDS §12.1 estimate; flagged if > +20%.

**Reports:**

- `reports/<commit>/<m>/synth_ooc.rpt`
- `reports/<commit>/<m>/timing_summary.rpt`
- `reports/<commit>/<m>/utilization_ooc.rpt`

**Sign-off criteria:** see [05-signoff-criteria.md](05-signoff-criteria.md) §6 (OOC synthesis sign-off).

---

## 8. Top-level synthesis gate

**Purpose:** prove the full RTL hierarchy + vendor IP synthesises with primary clocks, top-level XDC, and SLR placement intent.

**Command:** `make synth_top`

**What it runs:**

1. Generate output products for every IP under `prj/ip/` (cached).
2. `synth_design` against the full hierarchy with all `constraints/*.xdc` loaded in numeric order (§10).
3. `report_timing_summary`, `report_utilization`, `report_clocks`.

**Reports:** `reports/<commit>/top/synth.rpt`, `utilization.rpt`, `timing_summary.rpt`, `clocks.rpt`.

**Sign-off criteria:** see [05-signoff-criteria.md](05-signoff-criteria.md) §8 (top-level synthesis sign-off).

---

## 9. Implementation gate

**Purpose:** prove the design closes timing, meets utilisation and power budgets, and survives CDC and constraints sign-off.

**Command:** `make impl` (followed by `make timing`, `make cdc`)

**What it runs:**

1. `opt_design`, `place_design`, `phys_opt_design`, `route_design`, `phys_opt_design -post_route`.
2. `report_timing_summary`, `report_utilization`, `report_power`, `report_drc`, `report_cdc`, `check_timing`.
3. Bitstream generation (gated on clean reports).

**Reports:** `reports/<commit>/top/impl_timing.rpt`, `impl_utilization.rpt`, `impl_power.rpt`, `drc.rpt`, `cdc.rpt`, `check_timing.rpt`.

**Sign-off criteria:** see [05-signoff-criteria.md](05-signoff-criteria.md) §9 (implementation sign-off).

---

## 10. Constraints methodology

The constraints policy (file structure, naming, ownership rules, exception discipline) lives next to the artefacts:

- File structure and ownership → [../../constraints/README.md](../../constraints/README.md)
- Top-level vs module-internal vs OOC split → [../../constraints/README.md](../../constraints/README.md)
- Timing exception required fields → [../../constraints/README.md](../../constraints/README.md)

This section defines only the *gate* side: how the build invokes constraints checking, what reports are produced, and where waivers live.

**Build-side rules:**

1. XDC files load in numeric order (`00_clocks.xdc` → `07_debug.xdc`), then `ooc/<m>.xdc` for OOC contexts.
2. IP-generated XDC loads automatically before project XDC — never manually edited.
3. Every commit that adds or modifies a constraint must rerun `make timing` at the relevant stage (OOC or top).

**Timing exception waivers:** stored in `waivers/timing/`, one entry per exception, with the required fields listed in [../../waivers/README.md](../../waivers/README.md). A timing exception that exists in XDC but lacks a waiver entry fails `make timing` at the integration gate.

**Sign-off criteria:** see [05-signoff-criteria.md](05-signoff-criteria.md) §10 (constraints sign-off).

---

## 11. CDC methodology

CDC content is split: the design's actual CDCs live in FAD §4.5 (the inventory); the *gate* lives here.

**Approved CDC mechanisms** (per FAD §4.5; expanded as design progresses):

| Mechanism | When to use |
|---|---|
| Two-flop synchroniser | Single-bit asynchronous control / status |
| Async FIFO with Gray-code pointers | Multi-bit data with throughput |
| Handshake (req / ack) | Multi-bit data, low-rate |
| Bus-synchroniser (vendor IP) | Where vendor IP encapsulates the crossing |

**Forbidden:** ad-hoc multi-bit two-flop, gated/stretched-pulse synchronisers without a documented quasi-static contract.

**Command:** `make cdc MOD=<m>` (per module) and `make cdc` at top level.

**What it runs:**

1. Vivado `report_cdc` after elaboration / post-synthesis.
2. Third-party CDC tool (when adopted, per §2 baseline) at top-level for full-design analysis.
3. Cross-check report against FAD §4.5: every reported crossing must appear in the inventory; every inventory row must appear in the report.

**Reports:** `reports/<commit>/<m>/cdc.rpt`, `reports/<commit>/top/cdc.rpt`.

**CDC waivers:** stored in `waivers/cdc/`, one entry per waived violation, with required fields per [../../waivers/README.md](../../waivers/README.md). A waiver must cite the analysis (timing-quasi-static argument, custom synchroniser proof, etc.) that justifies it.

**Sign-off criteria:** see [05-signoff-criteria.md](05-signoff-criteria.md) §7 (CDC sign-off).

---

## 12. IP integration policy

The SFU integrates several vendor IPs (RFDC, Aurora 64B/66B, clocking wizard, AXI infrastructure). The policy below applies to every vendor IP.

### 12.1 IP source management

- Vendor IP configurations (`.xci` files) are version-controlled under `prj/ip/<ip_name>/`.
- IP **output products** (synthesised netlists, sim models, generated XDC) are produced by the build, not committed.
- IP version is pinned to the Vivado release in the tool baseline (§2). A Vivado upgrade implies an IP regeneration.

### 12.2 IP upgrade policy

- An IP version upgrade that changes the IP's interface, latency, resource usage, or simulation-model behaviour requires an ADR.
- Minor patch upgrades within the same Vivado release require regression but no ADR.
- After any IP upgrade: rerun `make sim_unit` for every wrapper module that consumes the IP, and `make synth_top` + `make timing` to validate at top level.

### 12.3 IP-generated constraints

- IP-generated XDC files are not manually edited.
- The Vivado flow loads IP constraints **before** project constraints — this ordering is fixed.
- Any project constraint that interacts with an IP-internal path must be documented in `05_timing_exceptions.xdc` with proof, per [../../constraints/README.md](../../constraints/README.md).

### 12.4 Simulation models

- Vendor-compiled simulation models are used for unit simulation of wrapper modules.
- Behavioural (not post-synthesis) IP models are used for OOC synthesis where applicable.
- Simulation-model compilation is part of `make sim_unit` and `make sim_all`.

### 12.5 Verification boundary

- Vendor IP internals are `not_applicable` for bit-exactness (MDS §3.3 / §7.3).
- The wrapper module (MDS reuse type = `wrap`) owns protocol and integration verification.
- The unit TB for a wrapper module verifies: parameterisation, port connectivity, protocol compliance at wrapper boundaries, reset behaviour, error reporting.

---

## 13. Waiver policy

Waivers are version-controlled under `waivers/` and reviewed at every gate they affect. The required fields and review discipline live in [../../waivers/README.md](../../waivers/README.md).

This section defines the *flow-side* expectations:

1. Every waiver entry has a unique ID (`WVR-<DOMAIN>-NNN`) referenced from the relevant report.
2. A waiver without an expiry trigger blocks integration.
3. Waivers are reviewed at the gate they affect (lint waiver → integration gate; CDC → CDC sign-off; timing → constraints sign-off + implementation sign-off).
4. The `make` target that runs the corresponding check loads the matching waiver file and only suppresses the listed entries — never silently.

---

## 14. Report archival

All tool-generated reports are archived under `reports/<commit-hash>/` so any sign-off can be reproduced from a committed state.

```text
reports/
└── <commit-hash>/
    ├── doc_check.log
    ├── <module_name>/
    │   ├── lint.log
    │   ├── unit_sim.xml
    │   ├── coverage.rpt
    │   ├── synth_ooc.rpt
    │   ├── timing_summary.rpt
    │   ├── cdc.rpt
    │   └── check_timing.rpt
    └── top/
        ├── synth.rpt
        ├── utilization.rpt
        ├── impl_timing.rpt
        ├── impl_power.rpt
        ├── cdc.rpt
        └── check_timing.rpt
```

The `reports/` folder is gitignored except for its README and `.gitignore`. CI archives selected runs to durable storage; locally, reports are regenerable.

The relevant MDS §12.3 row records the commit hash for each report consumed at sign-off.

---

## 15. CI regression levels

| Level | Trigger | Stages run | Wall-clock target |
|---|---|---|---|
| L0 | Pre-commit (local) | `doc_check`, `lint MOD=<changed>` | < 1 min |
| L1 | Push to feature branch | L0 + `sim_unit MOD=<changed>`, `synth_ooc MOD=<changed>`, `cdc MOD=<changed>` | < 15 min |
| L2 | PR open / update | L1 for every changed module | < 30 min |
| L3 | Nightly | L2 across all modules + `synth_top`, `cdc` (top), `impl`, `timing` | < 6 h |
| L4 | Release candidate | L3 + power report, full coverage roll-up, hardware bring-up smoke tests | < 12 h |

Each level builds on the previous — promotion to a higher level is gated on a clean lower-level run at the same commit.

---

## 16. Claude-assisted usage policy

Claude Code is used as a repo-aware operator for execution-flow tasks. Architect-side LLM workflow (drafting, reviewing) is defined in [02-architect-workflow.md](02-architect-workflow.md) §6 and is not duplicated here.

### 16.1 Claude Code execution-flow tasks

| Task | Example |
|---|---|
| Report summarisation | "Summarise the failures in `reports/<commit>/top/timing_summary.rpt` by clock domain." |
| Cross-document consistency | "Verify every CDC in `reports/<commit>/top/cdc.rpt` appears in FAD §4.5." |
| Waiver authoring | "Draft a `waivers/cdc/<file>.waiver` entry for CDC-2 violation X with citation pointing to ADR-NNNN." |
| Build-script edits | "Add a new `make` target `cdc_diff` that compares two CDC reports." |
| Pre-commit assistance | "Run `make doc_check` and propose minimal edits to fix every failure." |

### 16.2 What Claude Code does NOT do

- Sign off on any report, gate, or waiver.
- Modify frozen / baselined documents without explicit instruction.
- Silently resolve placeholder markers.
- Author or modify ADRs without architect direction.
- Push, force-push, or merge.

### 16.3 Anti-laundering discipline

The risk in LLM-assisted flows is plausible-but-wrong output. Countermeasures (also in [01-fpga-project-methodology.md](01-fpga-project-methodology.md) §5.3):

- Every report summary cites the underlying report file and line range.
- Every waiver draft cites the analysis or ADR it relies on.
- Every consistency-check finding cites the source-of-truth section it compared against.
- Engineer review is required before any flow gate accepts Claude-generated content.

---

## 17. References

- [01-fpga-project-methodology.md](01-fpga-project-methodology.md) — top-level methodology
- [02-architect-workflow.md](02-architect-workflow.md) — architect workflow and LLM-assisted drafting
- [03-rtl-design-workflow.md](03-rtl-design-workflow.md) — RTL designer workflow
- [05-signoff-criteria.md](05-signoff-criteria.md) — sign-off criteria, cross-referenced from each gate above
- [../../constraints/README.md](../../constraints/README.md) — XDC structure and ownership
- [../../waivers/README.md](../../waivers/README.md) — waiver discipline and required fields
- [../../arch/fad/_template.md](../../arch/fad/_template.md) — FAD template (clock inventory §4.1, CDC inventory §4.5, verification contract §12)
- UG949 — AMD UltraFast Design Methodology
- UG903 — AMD Vivado Using Constraints
- UG906 — AMD Vivado Design Analysis and Closure
- UG900 — AMD Vivado Logic Simulation

---

## 18. Open items

| ID | Item | Owner | Expiry trigger | Status |
|---|---|---|---|---|
| EXEC-FLOW-001 | Pin Vivado version, simulator, lint tool, CDC tool in §2 | Architect + RTL Designer | First top-level synthesis run | open |
| EXEC-FLOW-002 | Author `Makefile` and `scripts/lint/`, `scripts/sim/`, `scripts/synth/` skeletons | RTL Designer | First module reaches `design_ready` | open |
| EXEC-FLOW-003 | Author `make doc_check` script enforcing §4 rules | Architect | Templates stable (after Step 10) | open |
| EXEC-FLOW-004 | Define waiver file format (TOML / YAML / Spyglass-native) per tool choice | RTL Designer | Lint tool selected | open |
| EXEC-FLOW-005 | Define CI provider and regression-level wall-clock targets | Architect | Before L1 first run | open |
