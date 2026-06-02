## Overall assessment

**Score: 8.3 / 10**

This is a strong framework. It is much better than the usual “architecture.docx + random RTL + late constraints” flow. It is especially good for a Claude-assisted FPGA architecture workflow because it is Markdown-based, structured, diffable, greppable, and machine-checkable.

The biggest weakness is that it is still **documentation-heavy but flow-light**. It defines what documents exist and what they should contain, but it does not yet fully define the **engineering execution loop**: automated checks, report generation, constraints validation, CDC sign-off, timing baselining, OOC synthesis, implementation closure, hardware bring-up, and evidence capture.

In short:

```text
Your framework is strong as an architecture/specification framework.
It is weaker as a complete FPGA project methodology.
```

AMD’s UltraFast methodology is explicitly about efficient resource use, faster implementation convergence, and timing closure in Vivado, not only about documentation structure. Your proposal references UG949-style checks, but does not yet turn them into concrete Vivado/Tcl/CI/report gates. ([AMD Documentation][1])

---

# Strengths

## 1. Good document taxonomy

The 5-document structure is sensible:

```text
FAD  → device-level architecture
MDS  → module implementation spec
ICD  → shared interface contracts
ADR  → architectural decisions
RTM  → requirements traceability
```

That is lean enough to avoid document explosion but rich enough to support architecture, RTL, verification, and traceability.

The decision to reject a 15-document structure is correct. For a small team, too many documents create synchronization overhead and drift.

## 2. Strong separation between specified / inferred / assumed

Your placeholder discipline is one of the best parts:

```text
[TBD]
[STUB]
[ASSUMPTION]
[INFERRED]
```

That directly addresses a real failure mode in LLM-assisted engineering: the model silently converts inference into fact.

This is particularly important for BPMS/SFU, where system architecture, ownership boundaries, and band-level vs beam-level partitioning are load-bearing.

## 3. Good lifecycle model

The status model is useful:

```text
draft
in_review
baselined
frozen
superseded
rtl_ready
not_rtl_ready
```

The `rtl_ready_blocking` field is also good. It turns “not ready” into an actionable list rather than a vague state.

## 4. Strong module-level RTL handoff template

The MDS is very good. It includes most things an RTL engineer actually needs:

```text
ports
parameters
registers
clock/reset/CDC
fixed-point formats
pipeline
FSMs
FIFO/memory
error cases
verification scenarios
resource/latency estimates
```

The `rtl_ready` checklist is close to what I would use before allowing RTL generation or RTL coding to begin.

## 5. ICD-first approach is correct

Freezing core ICDs early is the right instinct.

For FPGA projects, interface ambiguity creates more rework than most algorithmic details. Your emphasis on streaming bus, register bus, and OBG frame ICDs is sound.

## 6. Good traceability mindset

The RTM is not overcomplicated. It links:

```text
requirement → FAD section → module → unit TB → UVM → evidence
```

That is the right structure.

## 7. Good debug/observability bias

You correctly make debug telemetry part of architecture, not an afterthought. That is important for hardware bring-up, especially with high-throughput DSP paths where waveform-level visibility is limited.

---

# Weaknesses

## 1. No explicit implementation-flow document

You have a documentation framework, but not yet a complete project execution flow.

Missing concrete flow:

```text
elaborate
lint
sim
synth_ooc
synth_top
opt_design
place_design
phys_opt_design
route_design
report_timing_summary
report_cdc
report_methodology
report_qor_suggestions
report_utilization
report_power
bitstream
hardware validation
```

AMD UG906 focuses heavily on analysis reports and closure techniques, including timing, clock trees, constraints, floorplanning, and runtime/result tradeoffs. Your framework mentions budgets, but does not yet define the Vivado report loop that proves the design is converging. ([AMD Documentation][2])

## 2. Constraints methodology is underdeveloped

You have:

```text
constraints/<module_name>.xdc
FAD clocking
FAD floorplan intent
```

But you do not yet define an XDC methodology.

Missing:

```text
constraints file ordering
clock constraints ownership
generated clocks
input/output delays
clock groups
CDC-related exceptions
false-path waiver discipline
multicycle-path proof requirements
constraint validation reports
OOC constraints vs top-level constraints
IP-generated constraints handling
```

UG903 is specifically about XDC constraints, including organizing constraints, project/non-project flows, OOC constraints, synthesis/implementation constraints, ordering, and recommended constraint sequencing. Your framework needs a first-class constraints section or checklist aligned with this. ([AMD Documentation][3])

## 3. CDC sign-off is specified but not operationalized

You have a CDC inventory. Good.

But missing:

```text
report_cdc requirement
waiver mechanism
CDC synchronizer library
allowed CDC patterns
forbidden CDC patterns
reconvergence policy
multi-bit CDC policy
async reset crossing policy
CDC ownership per module
```

A table saying “all CDCs must appear here” is not enough. You need automated detection and waiver control.

Add something like:

```text
cdc/
  allowed_patterns.md
  waivers.yaml
  report_cdc_summary.md
```

Or fold this into FAD + CI.

## 4. Verification is too thin for your stated goal

You deliberately made FAD §12 “thin” because DV owns UVM. That is reasonable organizationally, but risky architecturally.

The architecture side must still define:

```text
verification intent
architectural invariants
end-to-end scenarios
latency checks
throughput checks
numeric acceptance metrics
reset/recovery scenarios
stress scenarios
fault-injection requirements
hardware bring-up tests
```

Otherwise, the MDS can become RTL-ready without proving that the architecture is testable.

UG900 covers RTL behavioral simulation, post-synthesis simulation, post-implementation simulation, test benches, stimulus files, simulator setup, and simulation libraries. Your framework references unit TBs, but does not yet define when each simulation level is required. ([AMD Documentation][4])

## 5. No explicit CI/regression framework

You mention pre-commit hooks, but not the full automation flow.

Missing repo-level files/concepts:

```text
Makefile targets
Tcl entrypoints
CI pipeline stages
artifact naming
report extraction
pass/fail thresholds
nightly regression
seed management
test vector versioning
coverage merge
dashboard or summary report
```

For this project, I would expect at least:

```text
make lint
make sim_unit MODULE=...
make sim_all
make synth_ooc MODULE=...
make synth_top
make impl
make reports
make cdc
make timing
make rtm_check
make doc_check
```

Without this, the framework depends too much on discipline.

## 6. Resource and timing budgets need “measured evidence” linkage

You have pre-synthesis and post-synthesis tables. Good.

But missing:

```text
expected vs actual comparison
trend across commits
QoR regression threshold
timing slack threshold
logic-level threshold
fanout threshold
congestion threshold
SLR crossing count
DSP/BRAM/URAM packing efficiency
```

UG949 and UG906 are report-driven. The methodology should say which reports must be generated and what fails the gate. Timing closure is easier when RTL and constraints are designed correctly before implementation, and AMD frames design closure as meeting performance, timing, power, and hardware validation requirements. ([AMD Documentation][5])

## 7. Hardware bring-up is underrepresented

You have debug architecture, capture, playback, loopback, event log. Good.

But missing a formal bring-up sequence:

```text
power-on sanity
clock/PLL lock
reset release
management bus access
register readback
version ID check
Aurora/GT link bring-up
RFDC bring-up
1PPS/UTC validation
static pattern injection
loopback
capture validation
live OBG path validation
error injection
long-run soak
```

For an SFU-class design, this should exist somewhere. Not necessarily in FAD, but at least as a document or checklist.

## 8. No explicit AMD/Xilinx IP integration policy

This matters for SFU because you likely touch some combination of:

```text
Aurora
GT transceivers
RFDC
AXI infrastructure
clocking wizard / MMCM / PLL
FIFOs
BRAM/URAM
ILA/VIO
possibly NoC/PS if Versal/Zynq involved
```

Missing:

```text
IP configuration capture
.xci versioning policy
generated output policy
IP upgrade policy
simulation model compilation
IP constraints handling
black-box/vendor-IP verification boundary
```

UG900 explicitly covers simulation model/library setup for AMD device/IP simulation. Your framework has `third-party` and `vendor IP opaque`, but not a full IP methodology. ([AMD Documentation][6])

## 9. The FAD may become too large

Your FAD contains:

```text
scope
block diagram
dataflow
clock/reset/CDC
memory
module inventory
interface conventions
fixed-point policy
budgets
debug
management
verification
risks
glossary
```

That is acceptable at first. But for SFU, the FAD may become a monster.

You may eventually need extracted appendices while keeping FAD as the index:

```text
fad/FAD.md
fad/clock_reset_cdc.md
fad/numerical_policy.md
fad/debug_observability.md
fad/budgets.md
```

I would not split now. But monitor size.

## 10. “Claude Code can generate RTL” is too strong as a gate label

This line is risky:

```text
rtl_ready — every RTL-ready criterion met; Claude Code can generate RTL from this + referenced ICDs
```

Better:

```text
rtl_ready — sufficient for RTL implementation to start by an engineer or Claude-assisted flow
```

Do not define the success condition as “Claude can generate RTL.” Define it as “implementation-ready.” Claude is a tool, not the owner of correctness.

---

# Detailed gaps versus the methodology previously summarized

## A. Requirements

### What you have

Strong:

```text
RTM
source document citations
FAD functional boundary
MDS upstream requirements
specified / inferred / assumed markers
```

### Missing

You explicitly say:

```text
No separate HRD; requirements are inherited from SFU Arch Doc §18.
```

That is acceptable only if the inherited requirements are already precise enough.

Risk: system requirements often are not directly FPGA-implementable.

You need a category for derived FPGA requirements:

```text
FAD-DER-xxx
```

Examples:

```text
FAD-DER-CLK-001: All CDC from Aurora clock to DSP clock shall use async FIFO.
FAD-DER-LAT-003: Bins selector shall add no more than N cycles latency.
FAD-DER-DBG-002: Every module shall expose frame counters and sticky error flags.
```

Your RTM mentions derived requirements, but the process for creating and reviewing them is not explicit enough.

### Add

```text
Derived Requirement Policy
- Derived requirements are allowed.
- They must be tagged FAD-DER-*.
- Each must cite the parent requirement or architectural decision that forced it.
- Each must appear in RTM.
- Each must have at least one verification target.
```

---

## B. Architecture and micro-architecture

### What you have

Very strong:

```text
FAD
MDS
ICD
ADR
module inventory
pipeline tables
FSMs
fixed-point policy
debug architecture
management plane
```

### Missing

You need explicit **architecture invariants**.

Examples:

```text
No beam-level processing in SFU.
No unregistered ready/valid path across module boundaries.
No data loss under legal backpressure.
All UTC-scheduled parameter commits are atomic at frame boundary or 1PPS boundary.
All DSP modules preserve frame ordering.
All clock crossings are inventoried and implemented by approved CDC primitives.
```

These are higher-level than individual requirements. They become assertion targets and review checks.

### Add to FAD

```markdown
## Architecture Invariants

| ID | Invariant | Rationale | Checked by |
|---|---|---|---|
| INV-001 | SFU performs band-level processing only; no per-beam gain/doppler/delay. | Functional boundary | FAD review, RTM |
| INV-002 | No combinational ready/valid path crosses module boundary. | Timing closure | lint/SVA |
| INV-003 | All CDCs use approved mechanisms. | Hardware safety | report_cdc |
```

---

## C. Constraints

### What you have

Partial:

```text
Clock inventory
CDC inventory
floorplan intent
constraints/<module_name>.xdc
```

### Missing

This is the largest technical gap.

You need a constraints methodology section.

### Add to FAD or README

```markdown
## Constraints Methodology

### XDC file structure

constraints/
  00_clocks.xdc
  01_resets.xdc
  02_io.xdc
  03_generated_clocks.xdc
  04_clock_groups.xdc
  05_timing_exceptions.xdc
  06_physical.xdc
  07_debug.xdc
  ooc/
    <module_name>.xdc

### Required reports

- report_clocks
- report_clock_interaction
- report_cdc
- report_exceptions
- report_timing_summary
- report_methodology
- check_timing
```

### Add waiver policy

```text
False path:
- allowed only with named CDC/safe-static reason
- must cite MDS/FAD section
- must have owner and review date

Multicycle path:
- requires waveform/protocol proof
- setup and hold constraints must be paired
- must be regression checked
```

This is not optional. Bad XDC silently invalidates timing sign-off.

---

## D. Timing closure

### What you have

Partial:

```text
latency budget
resource budget
floorplan intent
critical path notes
target Fmax
post-synthesis actual
```

### Missing

You need a timing closure loop.

### Add

```markdown
## Timing Closure Methodology

### Per-module OOC gate
- elaborate clean
- synth_ooc clean
- utilization within budget
- estimated Fmax >= target + margin
- no unexpected inferred latches
- no unconstrained clocks
- no high-fanout control above threshold unless waived

### Top-level implementation gate
- WNS >= 0
- TNS = 0
- WHS >= 0
- no unconstrained paths
- no invalid/ignored timing exceptions
- congestion acceptable
- clock interaction clean or waived
```

### Add actual report references

```text
reports/<commit>/
  timing_summary.rpt
  methodology.rpt
  utilization.rpt
  cdc.rpt
  clock_interaction.rpt
  qor_suggestions.rpt
  power.rpt
```

This makes timing closure auditable.

---

## E. Verification

### What you have

Good unit-level structure:

```text
directed
constrained random
file-driven refmodel
coverage
assertions
error cases
backpressure
reset
scheduled apply
CDC stress
```

### Missing

You need an architecture-level verification matrix, even if DV owns UVM.

Add to FAD §12:

```text
Architectural Verification Matrix
```

Example:

| Feature               | Unit TB  | Subsystem | UVM/top | Hardware         |
| --------------------- | -------- | --------- | ------- | ---------------- |
| OBG frame alignment   | yes      | yes       | yes     | capture          |
| Bins selector mapping | yes      | yes       | yes     | playback/capture |
| Band gain             | bit-true | yes       | yes     | spectral check   |
| Band doppler          | bit-true | yes       | yes     | tone check       |
| UTC scheduled apply   | yes      | yes       | yes     | 1PPS test        |
| Backpressure          | yes      | yes       | yes     | stress           |
| Reset recovery        | yes      | yes       | yes     | bring-up         |

Also add:

```text
golden-vector policy
random seed policy
coverage waiver policy
post-synth sim policy
post-impl sim policy
```

---

## F. CI and reproducibility

### What you have

Only hints:

```text
Markdown in Git
pre-commit hook idea
Claude consistency checks
```

### Missing

You need concrete CI gates.

Add a repo-level methodology section:

```markdown
## Automation Gates

| Gate | Command | Pass criteria |
|---|---|---|
| Docs | make doc_check | no TBD in baselined docs |
| RTM | make rtm_check | all reqs mapped |
| Lint | make lint | no errors, waivers approved |
| Unit sim | make sim_unit | all pass |
| OOC synth | make synth_ooc | clean reports |
| CDC | make cdc | clean or waived |
| Top synth | make synth_top | utilization within budget |
| Impl | make impl | timing met |
| Reports | make reports | all reports archived |
```

This is where Claude can help safely: generate reports, summarize diffs, detect regressions. Not “trust Claude”; trust the tools.

---

## G. Hardware bring-up

### What you have

Good observability architecture.

### Missing

A bring-up plan.

Add either:

```text
docs/bringup_plan.md
```

or FAD §12.6:

```markdown
## Hardware Bring-up Sequence

1. Program FPGA and read `fw_version`.
2. Verify clock lock and reset release.
3. Verify management bus read/write.
4. Verify per-module `block_id`.
5. Verify static configuration mode.
6. Inject known playback pattern.
7. Capture at first tap.
8. Enable one processing block at a time.
9. Validate loopback path.
10. Validate live OBG input.
11. Validate UTC scheduled apply.
12. Run long-duration soak with counters checked.
```

For this kind of SFU design, this matters as much as simulation.

---

## H. AMD/Xilinx-specific methodology

### What you have

Some AMD-aware ideas:

```text
XDC
SLR/floorplan intent
RFdc adjacency
Vivado reports implied
UG949 referenced
```

### Missing

You need explicit AMD/Xilinx tool artifacts.

Add:

```text
Vivado version
target part
board part
IP catalog version
strategy used
synthesis options
implementation options
Tcl flow entrypoints
report set
XDC ordering
OOC IP generation policy
```

Example:

```markdown
## AMD/Xilinx Flow Baseline

| Item | Value |
|---|---|
| Vivado version | 2023.2 / 2025.x |
| Flow | non-project Tcl |
| Synthesis strategy | default / custom |
| Implementation strategy | Performance_Explore / custom |
| Timing signoff report | report_timing_summary |
| CDC signoff report | report_cdc |
| Methodology report | report_methodology |
| Power report | report_power |
```

UG903 notes XDC can be stored in XDC files or generated with Tcl, and AMD documents project and non-project constraint flows. Your framework should decide which one is authoritative. ([AMD Documentation][7])

---

# Specific edits I would make

## 1. Rename “Verification Strategy (thin)”

Current:

```text
## 12. Verification Strategy (thin)
```

Better:

```text
## 12. Architecture-Level Verification Contract
```

Because “thin” makes it sound optional or weak. The architecture contract can be concise, but it should not be weak.

## 2. Change this line

Current:

```text
rtl_ready — every RTL-ready criterion met; Claude Code can generate RTL from this + referenced ICDs
```

Better:

```text
rtl_ready — the module specification is sufficient for RTL implementation to start, either manually or with Claude-assisted code generation.
```

## 3. Add `implementation_ready` as separate from `rtl_ready`

For some modules, RTL can start before constraints, OOC, or verification hooks are final. You may want:

```text
spec_ready
rtl_ready
verification_ready
integration_ready
```

But do not overcomplicate unless needed.

A leaner option:

```text
rtl_ready
integration_ready
```

Where `integration_ready` requires:

```text
RTL done
unit TB passing
lint clean
OOC synth clean
CDC clean/waived
constraints present
resource/timing actuals recorded
```

## 4. Add report evidence to MDS

In MDS §12, add:

```markdown
### 12.3 Tool evidence

| Check | Report | Commit | Status |
|---|---|---|---|
| Lint | reports/<commit>/<module>/lint.log | | |
| Unit sim | reports/<commit>/<module>/sim.xml | | |
| OOC synth | reports/<commit>/<module>/synth_ooc.rpt | | |
| CDC | reports/<commit>/<module>/cdc.rpt | | |
| Timing | reports/<commit>/<module>/timing_summary.rpt | | |
```

## 5. Add SVA/assertion section to ICD

Your ICD defines protocol rules, but should explicitly require assertions.

Add:

```markdown
## Protocol Assertions

| Assertion | Condition |
|---|---|
| stable_while_stalled | if tvalid && !tready, payload and sideband remain stable |
| no_x_on_valid | when tvalid, payload and sideband are known |
| tlast_boundary | tlast only occurs at legal frame boundary |
| no_valid_during_reset | tvalid low during reset |
| no_comb_ready_valid_loop | checked structurally/lint where possible |
```

This is practical and valuable.

## 6. Add `doc_check` rules

Example:

```text
A baselined document fails doc_check if:
- it contains [STUB]
- it contains [TBD] without owner
- it contains [ASSUMPTION] without expiry trigger
- it references a non-existing MDS/ICD/ADR
- its source_docs are empty
- an MDS is rtl_ready but rtl_ready_blocking is non-empty
- an MDS is rtl_ready but has no verification scenarios
```

This converts your Markdown framework into an enforceable methodology.

---

# What is missing as document types?

You were right to avoid too many documents. But I would add **two lightweight repo-level files**, not full heavyweight specs:

## 1. `flow.md`

Purpose: define execution flow.

```text
tools
Vivado version
Tcl entrypoints
Make targets
CI gates
report locations
pass/fail criteria
waiver mechanism
```

## 2. `signoff.md`

Purpose: define what “done” means.

```text
doc sign-off
RTL sign-off
verification sign-off
CDC sign-off
constraints sign-off
timing sign-off
hardware bring-up sign-off
```

These are not architecture documents. They are project-methodology documents.

Your `arch/` folder can remain the spec source of truth.

Possible layout:

```text
docs/
  methodology/
    flow.md
    signoff.md
```

or:

```text
arch/
  README.md
  flow.md
  signoff.md
```

I would keep them in `docs/methodology/`, because they describe process, not the SFU architecture itself.

---

# Revised rating by category

| Area                        | Score | Comment                                                             |
| --------------------------- | ----: | ------------------------------------------------------------------- |
| Documentation structure     |   9.0 | Lean and coherent                                                   |
| Architecture capture        |   8.5 | Strong FAD/MDS/ICD split                                            |
| Requirements traceability   |   8.0 | Good RTM, needs derived-req policy                                  |
| Interface methodology       |   8.5 | Good ICDs, add protocol assertions                                  |
| Fixed-point/DSP methodology |   8.0 | Good, add metric-level acceptance criteria                          |
| Verification methodology    |   7.0 | Good unit TB content, weak architecture-level verification contract |
| Constraints methodology     |   5.5 | Biggest gap                                                         |
| Timing-closure methodology  |   6.0 | Budgets exist, closure loop missing                                 |
| CI/reproducibility          |   5.5 | Mentioned, not defined                                              |
| Hardware bring-up           |   6.0 | Debug architecture good, bring-up flow missing                      |
| AMD/Xilinx specificity      |   6.5 | Needs explicit Vivado/XDC/report/IP flow                            |

Overall: **8.3 / 10 as a documentation framework**; **7.0 / 10 as a full FPGA project methodology**.

---

# Bottom line

Your proposal is a solid architecture/spec framework. I would keep the 5 core document types.

The main correction is to add an execution layer:

```text
flow.md
signoff.md
CI/report gates
constraints methodology
CDC/report waiver policy
timing closure loop
hardware bring-up checklist
```

Do not create many more architecture documents. Instead, make the existing documents **executable** through checks, reports, and gates. That is the missing bridge between “good architecture notes” and “real AMD/Xilinx FPGA methodology.”

[1]: https://docs.amd.com/r/en-US/ug949-vivado-design-methodology?utm_source=chatgpt.com "UltraFast Design Methodology Guide for FPGAs and SoCs ..."
[2]: https://docs.amd.com/r/en-US/ug906-vivado-design-analysis?utm_source=chatgpt.com "Vivado Design Suite User Guide: Design Analysis and ..."
[3]: https://docs.amd.com/r/en-US/ug903-vivado-using-constraints?utm_source=chatgpt.com "Vivado Design Suite User Guide: Using Constraints (UG903)"
[4]: https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation?utm_source=chatgpt.com "Vivado Design Suite User Guide: Logic Simulation (UG900)"
[5]: https://docs.amd.com/r/en-US/ug949-vivado-design-methodology/Timing-Closure?utm_source=chatgpt.com "Timing Closure - 2025.1 English - UG949"
[6]: https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Preparing-for-Simulation?utm_source=chatgpt.com "Preparing for Simulation - 2025.2 English - UG900"
[7]: https://docs.amd.com/r/en-US/ug903-vivado-using-constraints/About-XDC-Constraints?utm_source=chatgpt.com "About XDC Constraints - 2025.2 English - UG903"
