Yes, clear.

You are asking for a **practical revision plan**, not another review. The goal is to modify your current `arch/` framework so it becomes both:

```text
architecture/specification framework
+
execution/sign-off methodology
```

I would do it in two layers:

```text
Layer 1 — patch the existing proposal
Layer 2 — add a lightweight execution layer
```

---

# Proposed plan

## 1. Keep the 5 core document types

Do **not** add many new architecture documents.

Keep:

```text
FAD
MDS
ICD
ADR
RTM
```

These are already good.

The architecture folder remains:

```text
arch/
├── README.md
├── fad/
├── modules/
├── icd/
├── adr/
└── rtm.md
```

## 2. Add two lightweight methodology files

Add:

```text
docs/methodology/
├── flow.md
└── signoff.md
```

Reason: these are process/execution documents, not SFU architecture specs.

Alternative if you want everything together:

```text
arch/
├── flow.md
└── signoff.md
```

My recommendation: **use `docs/methodology/`**.

---

# Concrete changes to the original proposal

## Change 1 — update `README.md`

Add a new section:

```markdown
## Execution Layer

The `arch/` folder defines what must be built. The execution layer defines how the design is implemented, checked, reported, and signed off.

Execution documents live in:

`docs/methodology/`

- `flow.md` — tool flow, automation gates, Vivado/Tcl/CI commands, report locations
- `signoff.md` — sign-off criteria for docs, RTL, CDC, constraints, timing, verification, and hardware bring-up

No FAD, MDS, ICD, or RTM item is considered complete only because the Markdown is complete. Completion requires evidence from the execution layer.
```

## Change 2 — rename FAD §12

Current:

```markdown
## 12. Verification Strategy (thin)
```

Replace with:

```markdown
## 12. Architecture-Level Verification Contract
```

This section should not be huge, but it must define what the architecture expects DV, unit TBs, and hardware bring-up to prove.

## Change 3 — add architecture invariants to FAD

Add after FAD §1.4 or after §3:

```markdown
## Architecture Invariants

| ID | Invariant | Rationale | Checked by |
|---|---|---|---|
| INV-001 | SFU performs band-level processing only; beam-level gain, Doppler, and delay remain outside the SFU unless a boundary-change ADR is accepted. | Preserves functional ownership. | FAD review, RTM |
| INV-002 | All inter-module streaming interfaces obey the frozen streaming ICD. | Prevents protocol drift. | ICD review, lint/SVA, unit TB |
| INV-003 | No unregistered combinational ready/valid path crosses a module boundary. | Timing closure and composability. | RTL review, lint/SVA |
| INV-004 | Every CDC is listed in FAD §4.5 and implemented using an approved CDC mechanism. | Hardware safety. | CDC review, `report_cdc` |
| INV-005 | Scheduled configuration updates commit atomically at the defined trigger boundary. | Prevents mixed-parameter operation. | Unit TB, UVM, hardware test |
| INV-006 | DSP modules with non-trivial numerical behavior have a bit-true or explicitly bounded reference model. | Numerical correctness. | Unit TB, refmodel correlation |
```

## Change 4 — add derived requirements policy to RTM or README

Add to `rtm.md`:

```markdown
## Derived Requirement Policy

Derived requirements are allowed when the FPGA architecture must introduce implementation requirements not stated literally in the parent system documents.

Rules:

1. Derived requirements use ID format `FAD-DER-<CATEGORY>-NNN`.
2. Each derived requirement must cite the parent requirement, ICD, ADR, or FAD section that forced it.
3. Each derived requirement must appear in the RTM.
4. Each derived requirement must map to at least one verification target.
5. Derived requirements cannot silently alter SFU functional ownership; boundary changes require an ADR.
```

## Change 5 — strengthen ICDs with protocol assertions

Add to every ICD template:

```markdown
## Protocol Assertions

| Assertion ID | Rule | Checked by |
|---|---|---|
| IF-SVA-001 | Payload and sideband remain stable while `valid && !ready`. | SVA / unit TB |
| IF-SVA-002 | `valid` is deasserted during reset and for the required reset recovery window. | SVA / unit TB |
| IF-SVA-003 | No unknown values on payload, sideband, or control when `valid` is high. | SVA / simulation |
| IF-SVA-004 | `last` occurs only at legal frame boundaries. | SVA / scoreboard |
| IF-SVA-005 | No sample is dropped or reordered under legal backpressure. | scoreboard |
```

## Change 6 — add execution-evidence section to MDS

Add to MDS after §12:

```markdown
## 13. Execution Evidence

| Check | Required before | Report / Evidence | Commit | Status |
|---|---|---|---|---|
| Lint | `integration_ready` | `reports/<commit>/<module>/lint.log` | | |
| Unit simulation | `integration_ready` | `reports/<commit>/<module>/unit_sim.xml` | | |
| Coverage | `integration_ready` | `reports/<commit>/<module>/coverage.rpt` | | |
| OOC synthesis | `integration_ready` | `reports/<commit>/<module>/synth_ooc.rpt` | | |
| OOC timing | `integration_ready` | `reports/<commit>/<module>/timing_summary.rpt` | | |
| CDC | `integration_ready` | `reports/<commit>/<module>/cdc.rpt` | | |
| Constraints check | `integration_ready` | `reports/<commit>/<module>/check_timing.rpt` | | |
```

Then renumber current MDS §13 and §14 to §14 and §15.

## Change 7 — add `integration_ready`

Keep `rtl_ready`, but add a stronger post-RTL status.

In README:

```markdown
MDSs additionally carry one of:

- **not_rtl_ready** — at least one criterion unmet
- **rtl_ready** — sufficient for RTL implementation to start
- **integration_ready** — RTL exists and has passed unit-level execution gates: lint, unit simulation, OOC synthesis, CDC, constraints sanity, and resource/timing report capture
```

This prevents confusing “ready to code” with “ready to integrate”.

## Change 8 — soften Claude wording

Replace:

```text
Claude Code can generate RTL
```

with:

```text
RTL implementation can start, either manually or with Claude-assisted code generation.
```

This keeps Claude in the loop without making it the correctness authority.

---

# Add the execution layer

Create:

```text
docs/methodology/flow.md
docs/methodology/signoff.md
```

Below is a first iteration template you can use directly.

---

# `docs/methodology/flow.md`

````markdown
---
doc_id: <project>-FLOW-001
doc_type: methodology
project: <project>
status: draft
version: 0.1
date: YYYY-MM-DD
author: <name>
toolchain:
  vendor: AMD/Xilinx
  vivado_version: <YYYY.x>
  vitis_version: <YYYY.x or N/A>
  simulator: <xsim / questa / vcs / verilator / other>
  flow_type: <non_project_tcl / project_mode>
---

# FPGA Execution Flow

This document defines the executable engineering flow for the SFU FPGA project.

The architecture documents define **what** must be built. This document defines **how** the design is checked, implemented, reported, and made reproducible.

---

## 1. Scope

### 1.1 In scope

- Documentation checks
- RTL lint
- Unit simulation
- Reference-model correlation
- Out-of-context synthesis
- Top-level synthesis
- Implementation
- CDC reporting
- Timing reporting
- Utilization reporting
- Power reporting
- Report archival
- CI / regression gates

### 1.2 Out of scope

- Detailed UVM test plan owned by DV
- Board-level lab procedure, except for links to bring-up checklist
- System-level validation outside the FPGA repository

---

## 2. Repository assumptions

```text
bpms-sfu-fpga-design/
├── arch/
├── docs/
│   └── methodology/
├── rtl/
├── model/
├── constraints/
├── scripts/
├── prj/
├── tb/
└── reports/
````

The verification repository consumes `arch/` as a read-only reference.

---

## 3. Tool baseline

| Item             | Value                           |
| ---------------- | ------------------------------- |
| Vivado version   | `<TBD>`                         |
| Vitis version    | `<TBD / N/A>`                   |
| Simulator        | `<TBD>`                         |
| Lint tool        | `<TBD>`                         |
| CDC tool         | `Vivado report_cdc` initially   |
| Flow type        | `<non-project Tcl recommended>` |
| Target device    | `<part number>`                 |
| Board / platform | `<board or custom>`             |

Tool version changes require an ADR if they affect timing, IP generation, simulation models, or reproducibility.

---

## 4. Build entrypoints

All repeatable flows must be callable from the command line.

| Command                          | Purpose                                                             |
| -------------------------------- | ------------------------------------------------------------------- |
| `make doc_check`                 | Validate Markdown metadata, links, TBD/STUB policy, RTM consistency |
| `make lint`                      | Run RTL lint                                                        |
| `make sim_unit MODULE=<module>`  | Run one module unit testbench                                       |
| `make sim_all`                   | Run all unit simulations                                            |
| `make synth_ooc MODULE=<module>` | Run out-of-context synthesis for one module                         |
| `make synth_top`                 | Run top-level synthesis                                             |
| `make impl`                      | Run implementation                                                  |
| `make cdc`                       | Run CDC reports                                                     |
| `make timing`                    | Run timing reports                                                  |
| `make reports`                   | Generate report bundle                                              |
| `make clean_reports`             | Remove generated reports                                            |

If a command is not implemented yet, mark it `[STUB: command not yet implemented, <owner>]`.

---

## 5. Documentation check gate

### 5.1 Command

```bash
make doc_check
```

### 5.2 Checks

A baselined document fails if:

* YAML frontmatter is missing.
* `doc_id`, `doc_type`, `status`, `version`, or `source_docs` are missing.
* It contains `[STUB:]`.
* It contains `[TBD:]` without owner.
* It contains `[ASSUMPTION:]` without expiry trigger.
* It references a non-existing FAD/MDS/ICD/ADR.
* An MDS has `status: rtl_ready` but `rtl_ready_blocking` is non-empty.
* An MDS has `status: rtl_ready` but no verification scenarios.
* An RTM row has no module or no verification target unless explicitly waived.
* A requirement is marked `closed` without evidence.

### 5.3 Output

```text
reports/<commit>/doc_check/doc_check.rpt
```

---

## 6. RTL lint gate

### 6.1 Command

```bash
make lint
```

### 6.2 Minimum checks

* No syntax errors.
* No inferred latches unless waived.
* No unintended combinational loops.
* No width truncation without explicit cast/comment.
* No undriven outputs.
* No multiple drivers.
* No unused major logic unless waived.
* Reset style matches MDS.
* Module/interface names match MDS.

### 6.3 Waivers

Lint waivers must be committed under:

```text
waivers/lint/
```

Each waiver must include:

* warning ID
* file/line or object
* reason
* owner
* expiry trigger

### 6.4 Output

```text
reports/<commit>/lint/lint.rpt
```

---

## 7. Unit simulation gate

### 7.1 Command

```bash
make sim_unit MODULE=<module>
make sim_all
```

### 7.2 Minimum scenarios

Each module must cover the scenarios listed in its MDS §11.

Required baseline scenarios:

* nominal operation
* boundary values
* legal backpressure
* reset mid-operation
* all error conditions
* mode transitions
* scheduled apply, if applicable
* CDC stress, if applicable

### 7.3 Reference-model correlation

For `bit_exact` modules:

* expected output must come from the reference model
* tolerance = 0 LSB unless MDS says otherwise

For `ulp_bounded` modules:

* tolerance = MDS `ulp_bound`

For `not_applicable` modules:

* scoreboard checks protocol and control behavior

### 7.4 Output

```text
reports/<commit>/sim/<module>/unit_sim.xml
reports/<commit>/sim/<module>/coverage.rpt
```

---

## 8. Out-of-context synthesis gate

### 8.1 Command

```bash
make synth_ooc MODULE=<module>
```

### 8.2 Pass criteria

* Synthesis completes without errors.
* No unexpected inferred latch.
* No unexpected black box.
* Resource use is within MDS §12 estimate or variance is explained.
* Timing estimate meets target Fmax margin or is marked as a risk.
* Module-specific XDC is accepted without critical warnings.

### 8.3 Required reports

```tcl
report_utilization
report_timing_summary
report_methodology
check_timing
```

### 8.4 Output

```text
reports/<commit>/synth_ooc/<module>/utilization.rpt
reports/<commit>/synth_ooc/<module>/timing_summary.rpt
reports/<commit>/synth_ooc/<module>/methodology.rpt
reports/<commit>/synth_ooc/<module>/check_timing.rpt
```

---

## 9. Constraints methodology

### 9.1 XDC organization

```text
constraints/
├── 00_clocks.xdc
├── 01_generated_clocks.xdc
├── 02_resets.xdc
├── 03_io.xdc
├── 04_clock_groups.xdc
├── 05_timing_exceptions.xdc
├── 06_physical.xdc
├── 07_debug.xdc
└── ooc/
    └── <module_name>.xdc
```

### 9.2 Rules

* Clock definitions belong in `00_clocks.xdc`.
* Generated clocks belong in `01_generated_clocks.xdc`.
* IO delays belong in `03_io.xdc`.
* Clock-domain relationships belong in `04_clock_groups.xdc`.
* False paths and multicycle paths belong only in `05_timing_exceptions.xdc`.
* Physical constraints belong in `06_physical.xdc`.
* Debug core constraints belong in `07_debug.xdc`.
* IP-generated constraints are not edited manually.

### 9.3 Timing exception policy

Every false path or multicycle path must have:

* reason
* owner
* source/destination objects
* proof reference
* expiry trigger
* review status

Timing exceptions without documented proof are not allowed in baselined builds.

### 9.4 Required reports

```tcl
report_clocks
report_clock_interaction
report_exceptions
check_timing
report_timing_summary
```

---

## 10. CDC methodology

### 10.1 Approved CDC mechanisms

| CDC type           | Approved mechanism                        |
| ------------------ | ----------------------------------------- |
| single-bit control | 2-flop synchronizer                       |
| pulse              | pulse synchronizer or toggle synchronizer |
| multi-bit data     | async FIFO                                |
| counter            | Gray-coded counter                        |
| config bus         | shadow register + handshake               |
| status bus         | synchronizer or sampled status bridge     |

### 10.2 Forbidden unless waived

* Unsynchronized multi-bit CDC
* Reconvergent synchronized bits without encoding proof
* Async reset deassertion without synchronizer
* CDC hidden inside ad-hoc logic
* Combinational logic between synchronizer stages

### 10.3 Command

```bash
make cdc
```

### 10.4 Required report

```tcl
report_cdc
```

### 10.5 Waivers

CDC waivers live under:

```text
waivers/cdc/
```

Each waiver must include:

* CDC path
* mechanism
* reason
* owner
* expiry trigger
* reviewer

---

## 11. Top-level synthesis gate

### 11.1 Command

```bash
make synth_top
```

### 11.2 Pass criteria

* Synthesis completes.
* No unexpected black boxes.
* No unconstrained clocks.
* No major methodology violations.
* Resource use within FAD §9 budget or variance explained.
* All module-level generated reports archived.

### 11.3 Required reports

```tcl
report_utilization
report_timing_summary
report_methodology
check_timing
report_cdc
```

---

## 12. Implementation gate

### 12.1 Command

```bash
make impl
```

### 12.2 Pass criteria

* `place_design` completes.
* `route_design` completes.
* WNS >= 0 ns.
* TNS = 0 ns.
* WHS >= 0 ns.
* THS = 0 ns.
* No unconstrained paths.
* No critical methodology violations.
* Congestion is acceptable or documented.
* Power is within budget or documented.

### 12.3 Required reports

```tcl
report_timing_summary
report_timing
report_utilization
report_route_status
report_clock_utilization
report_clock_interaction
report_cdc
report_methodology
report_qor_suggestions
report_power
```

### 12.4 Output

```text
reports/<commit>/impl/timing_summary.rpt
reports/<commit>/impl/timing.rpt
reports/<commit>/impl/utilization.rpt
reports/<commit>/impl/route_status.rpt
reports/<commit>/impl/clock_utilization.rpt
reports/<commit>/impl/clock_interaction.rpt
reports/<commit>/impl/cdc.rpt
reports/<commit>/impl/methodology.rpt
reports/<commit>/impl/qor_suggestions.rpt
reports/<commit>/impl/power.rpt
```

---

## 13. Report archival

Every report bundle uses commit-based naming:

```text
reports/<git_commit>/<stage>/<module_or_top>/<report>.rpt
```

If the build is not from a clean Git tree, append:

```text
-dirty
```

Example:

```text
reports/8f3a21c-dirty/synth_ooc/bins_selector/timing_summary.rpt
```

---

## 14. CI / regression levels

| Level | Trigger            | Commands                                   | Purpose                   |
| ----- | ------------------ | ------------------------------------------ | ------------------------- |
| L0    | pre-commit         | `doc_check`, formatting                    | catch metadata/doc errors |
| L1    | pull request       | `lint`, selected unit sims                 | catch local RTL breakage  |
| L2    | nightly            | `sim_all`, `synth_ooc`                     | catch module regressions  |
| L3    | weekly / milestone | `synth_top`, `impl`, full reports          | project convergence       |
| L4    | release candidate  | full sim + full impl + hardware smoke test | sign-off candidate        |

---

## 15. Claude-assisted usage

Claude may assist with:

* filling document templates
* checking consistency across FAD/MDS/ICD/RTM
* generating draft Tcl scripts
* summarizing reports
* comparing QoR across commits
* drafting waivers
* proposing test scenarios

Claude must not be treated as sign-off authority. Sign-off evidence comes from:

* source review
* simulation results
* coverage reports
* Vivado reports
* hardware validation evidence

---

## 16. Open items

| ID          | Item                                      | Owner     | Expiry trigger                       | Status |
| ----------- | ----------------------------------------- | --------- | ------------------------------------ | ------ |
| FLOW-OI-001 | Select lint tool.                         | `<owner>` | Before first RTL PR                  | open   |
| FLOW-OI-002 | Decide simulator for unit TBs.            | `<owner>` | Before first MDS reaches `rtl_ready` | open   |
| FLOW-OI-003 | Implement `make doc_check`.               | `<owner>` | Before first baselined MDS           | open   |
| FLOW-OI-004 | Implement Vivado non-project Tcl scripts. | `<owner>` | Before first OOC synth gate          | open   |

````

---

# `docs/methodology/signoff.md`

```markdown
---
doc_id: <project>-SIGNOFF-001
doc_type: methodology
project: <project>
status: draft
version: 0.1
date: YYYY-MM-DD
author: <name>
---

# FPGA Sign-off Methodology

This document defines what "done" means for the SFU FPGA project.

A feature, module, or build is not signed off by document completion alone. Sign-off requires reviewed specifications and execution evidence.

---

## 1. Sign-off levels

| Level | Meaning |
|---|---|
| Documentation sign-off | Architecture/spec documents are complete and reviewed |
| RTL implementation sign-off | RTL is implemented and passes local checks |
| Module integration sign-off | Module is ready to integrate into top-level |
| Subsystem sign-off | Connected subsystem passes functional and implementation checks |
| FPGA implementation sign-off | Full FPGA build meets timing, constraints, CDC, utilization, and power gates |
| Hardware bring-up sign-off | Bitstream passes lab smoke and bring-up tests |

---

## 2. Documentation sign-off

### 2.1 FAD sign-off criteria

- Scope and functional boundary complete.
- Parent documents cited.
- Top-level block diagram complete.
- Dataflow complete.
- Clock/reset/CDC inventory complete.
- Module inventory complete.
- Interface conventions complete.
- Fixed-point policy complete.
- Latency/resource/floorplan/power budgets populated or explicitly TBD with owner.
- Debug and management architecture complete.
- Architecture-level verification contract complete.
- Open issues have owner and expiry trigger.
- `doc_check` passes.

### 2.2 MDS sign-off criteria

- Module purpose and exclusions complete.
- Interfaces resolved and linked to frozen or baselined ICDs.
- Parameters have type, default, and legal range.
- Clock/reset/CDC behavior complete.
- Fixed-point behavior complete.
- Pipeline/FSM/memory structure complete.
- Error and edge cases complete.
- Unit verification scenarios complete.
- Resource and latency estimates complete.
- Traceability to requirements complete.
- `rtl_ready_blocking` is empty for `rtl_ready`.

### 2.3 ICD sign-off criteria

- Signal list complete.
- Payload format complete.
- Sideband fields complete.
- Timing and backpressure rules complete.
- Error behavior complete.
- Protocol assertions listed.
- Versioning/change policy complete.
- All consumers listed.

### 2.4 RTM sign-off criteria

- Every inherited requirement has a row.
- Every derived requirement has a row.
- Every row maps to FAD section.
- Every implementation requirement maps to at least one module.
- Every closed requirement has verification evidence.
- Waived requirements cite waiver/ADR.

---

## 3. RTL-ready sign-off

A module may be marked `rtl_ready` when:

- MDS documentation sign-off is complete.
- Required ICDs are baselined or frozen.
- Required ADRs are accepted.
- Reference model exists if required.
- Unit TB strategy is defined.
- No `[STUB:]` remains.
- No unresolved `[TBD:]` affects RTL behavior.
- `rtl_ready_blocking` is empty.

`rtl_ready` means implementation may start. It does not mean the module is integration-ready.

---

## 4. RTL implementation sign-off

A module passes RTL implementation sign-off when:

- RTL source exists.
- RTL matches the MDS module name, ports, parameters, and reset conventions.
- RTL compiles/elaborates.
- Lint passes or all waivers are approved.
- Basic directed unit tests pass.
- Required protocol assertions are present or bound.
- No known mismatch exists between RTL and MDS.

Evidence:

```text
reports/<commit>/lint/<module>/lint.rpt
reports/<commit>/sim/<module>/unit_sim.xml
````

---

## 5. Module integration-ready sign-off

A module may be marked `integration_ready` when:

* RTL implementation sign-off is complete.
* Full unit TB scenarios from MDS §11 pass.
* Coverage targets are met or waived.
* Reference-model correlation passes.
* OOC synthesis passes.
* OOC timing report is generated.
* OOC utilization report is generated.
* CDC report is clean or waived.
* Module constraints are present and accepted.
* MDS §13 Execution Evidence table is updated.

Evidence:

```text
reports/<commit>/sim/<module>/
reports/<commit>/synth_ooc/<module>/
reports/<commit>/cdc/<module>/
```

---

## 6. Constraints sign-off

Constraints pass sign-off when:

* All clocks are defined.
* Generated clocks are defined.
* IO constraints are present where applicable.
* Clock interaction report is clean or waived.
* `check_timing` reports no unconstrained paths.
* False paths are justified.
* Multicycle paths include setup and hold intent.
* Timing exceptions are reviewed.
* Physical constraints match FAD floorplan intent.
* IP-generated XDC files are included in the correct order.

Evidence:

```text
reports/<commit>/impl/check_timing.rpt
reports/<commit>/impl/clock_interaction.rpt
reports/<commit>/impl/exceptions.rpt
```

---

## 7. CDC sign-off

CDC passes sign-off when:

* Every CDC is listed in FAD §4.5 or owning MDS.
* Every CDC uses an approved mechanism.
* `report_cdc` is clean or all findings are waived.
* No unsynchronized multi-bit CDC exists.
* Async resets deassert synchronously in each destination domain.
* CDC waivers have owner, reason, expiry trigger, and reviewer.

Evidence:

```text
reports/<commit>/impl/cdc.rpt
waivers/cdc/
```

---

## 8. Timing sign-off

Timing passes sign-off when:

* Implementation completes.
* WNS >= 0 ns.
* TNS = 0 ns.
* WHS >= 0 ns.
* THS = 0 ns.
* No unconstrained paths.
* No critical methodology violations.
* Worst paths are reviewed.
* High-fanout nets are reviewed.
* Congestion is acceptable or documented.
* Timing exceptions are reviewed.

Evidence:

```text
reports/<commit>/impl/timing_summary.rpt
reports/<commit>/impl/timing.rpt
reports/<commit>/impl/methodology.rpt
reports/<commit>/impl/qor_suggestions.rpt
```

---

## 9. Resource and power sign-off

Resource and power pass sign-off when:

* Utilization is within FAD budget or variance is approved.
* DSP/BRAM/URAM use matches architectural intent.
* No unexpected memory or DSP inference issue exists.
* Power estimate is within budget or risk is accepted.
* Thermal impact is reviewed if power exceeds budget.

Evidence:

```text
reports/<commit>/impl/utilization.rpt
reports/<commit>/impl/power.rpt
```

---

## 10. FPGA implementation sign-off

A full FPGA build is sign-off candidate when:

* Documentation sign-off complete.
* All integrated modules are `integration_ready`.
* Constraints sign-off complete.
* CDC sign-off complete.
* Timing sign-off complete.
* Resource and power sign-off complete.
* Top-level simulations required for the milestone pass.
* RTM evidence updated.
* Known issues are either closed or waived.

---

## 11. Hardware bring-up sign-off

### 11.1 Smoke test

A bitstream passes hardware smoke test when:

* FPGA programs successfully.
* Firmware version is readable.
* All expected module `block_id`s are readable.
* Clock lock/status registers are valid.
* Reset release is confirmed.
* Management read/write path works.
* Error counters are zero after reset.

### 11.2 Functional bring-up

Functional bring-up passes when:

* Static playback injection works.
* Capture path works.
* One known pattern propagates through the expected pipeline.
* Loopback mode works if applicable.
* UTC/1PPS scheduled apply is validated.
* At least one live data path is validated.
* Error injection produces expected telemetry.
* Long-run soak does not produce unexpected counter increments.

Evidence:

```text
reports/<commit>/hw_bringup/
```

Suggested artifacts:

```text
hw_bringup_log.md
register_dump_before.txt
register_dump_after.txt
capture_samples.bin
spectrum_check.png
known_issues.md
```

---

## 12. Waiver policy

Waivers are allowed but must be visible.

Every waiver must include:

* ID
* type: lint / CDC / timing / coverage / requirement / methodology
* object/path/test
* reason
* owner
* reviewer
* expiry trigger
* status
* link to evidence

Waivers must not be hidden in email or chat.

---

## 13. Release candidate checklist

A release candidate requires:

* [ ] FAD baselined
* [ ] Relevant MDSs baselined
* [ ] Relevant ICDs frozen
* [ ] RTM updated
* [ ] All modules `integration_ready`
* [ ] Full simulation regression passed
* [ ] Top-level synthesis passed
* [ ] Implementation passed
* [ ] Constraints sign-off passed
* [ ] CDC sign-off passed
* [ ] Timing sign-off passed
* [ ] Resource/power sign-off passed
* [ ] Bitstream generated
* [ ] Hardware smoke test passed
* [ ] Known issues documented
* [ ] Waivers reviewed

---

## 14. Claude-assisted usage

Claude may help prepare sign-off summaries by reading:

* FAD
* MDSs
* ICDs
* RTM
* Vivado reports
* simulation reports
* waiver files
* hardware bring-up logs

Claude-generated sign-off summaries are advisory. Sign-off decisions remain with the responsible engineer/reviewer.

---

## 15. Open items

| ID             | Item                                            | Owner     | Expiry trigger                        | Status |
| -------------- | ----------------------------------------------- | --------- | ------------------------------------- | ------ |
| SIGNOFF-OI-001 | Define exact coverage targets per module class. | `<owner>` | Before first integration-ready module | open   |
| SIGNOFF-OI-002 | Define hardware smoke-test register script.     | `<owner>` | Before first bitstream                | open   |
| SIGNOFF-OI-003 | Define waiver YAML format.                      | `<owner>` | Before first waiver                   | open   |

````

---

# Recommended next implementation sequence

Do it in this order:

## Step 1 — patch the wording

Change:

```text
Verification Strategy (thin)
````

to:

```text
Architecture-Level Verification Contract
```

Change:

```text
Claude Code can generate RTL
```

to:

```text
RTL implementation can start, either manually or with Claude-assisted code generation.
```

## Step 2 — add `integration_ready`

Add it to README lifecycle and MDS frontmatter.

## Step 3 — add FAD architecture invariants

This is high value and low effort.

## Step 4 — add MDS execution evidence

This connects specs to actual tool output.

## Step 5 — add ICD protocol assertions

This makes ICDs verifiable, not just descriptive.

## Step 6 — add `flow.md`

This is the main execution layer.

## Step 7 — add `signoff.md`

This defines “done”.

## Step 8 — later automate

Only after the templates are stable:

```text
make doc_check
make rtm_check
make report_summary
```

Do not start with automation. First stabilize the rules.

---

# Minimal first iteration

For the first iteration, I would implement only these files/changes:

```text
arch/README.md              # add execution layer + integration_ready
arch/fad/_template.md       # add architecture invariants + renamed verification section
arch/modules/_template.md   # add execution evidence + integration_ready language
arch/icd/_template.md       # add protocol assertions
arch/rtm.md                 # add derived requirement policy
docs/methodology/flow.md    # new
docs/methodology/signoff.md # new
```

That is enough to convert your framework from “good documentation” to a practical AMD/Xilinx-oriented FPGA methodology.
