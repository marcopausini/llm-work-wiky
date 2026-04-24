# FPGA Design Workflow — Industry Best Practice

**Audience:** FPGA architect / senior DSP engineer
**Lifecycle:** V-Model, tailored for FPGA development
**Normative references:** ECSS-Q-ST-60-02C (space) · DO-254 (avionics) · IEEE 1012 (V&V) · Accellera UVM (verification)

---

## 1. The V-Model at a Glance

```
 SPECIFICATION                                          VALIDATION
 ──────────────                                         ──────────
                                                            ▲
 ① Concept / Mission ────────────────────────────► ⑩ Acceptance
   │                                                        ▲
   ▼                                                        │
 ② System Architecture ───────────────────────────► ⑨ FPGA Validation (HW)
   │                                                        ▲
   ▼                                                        │
 ③ FPGA Requirements ─────────────────────────────► ⑧ Integration Verification
   │                                                        ▲
   ▼                                                        │
 ④ FPGA Architecture (HLD) ───────────────────────► ⑦ Module / Unit Verification
   │                                                        ▲
   ▼                                                        │
 ⑤ Detailed Design (LLD) ─────────┐          ┌─────────────┘
                                    ▼          ▲
                                   ⑥ Implementation (RTL / HLS / P&R)

    LEFT SIDE: decomposition          RIGHT SIDE: integration
```

**Key terminology distinction:**

| Term | Question it answers | Against what |
|------|--------------------|--------------|
| **Verification** | *Did we build the thing right?* | Against requirements / specification |
| **Validation** | *Did we build the right thing?* | Against mission need, in real HW |

---

## 2. Phase-by-Phase Breakdown

### Left Side of the V — Specification & Design

#### ① Concept / Mission Analysis

| | |
|---|---|
| **Purpose** | Establish *why* the FPGA exists in the system |
| **Scope** | Stakeholder needs, operational scenarios, technology baseline |
| **Exit review** | Mission Concept Review (MCR) |

**Artifacts**

- Concept of Operations (**ConOps**)
- Mission / System Requirements Document (**MRD / SyRS**)
- Technology trade studies
- Feasibility / prototyping reports

---

#### ② System Architecture & Partitioning

| | |
|---|---|
| **Purpose** | Decide HW/SW split, device class, and criticality |
| **Scope** | FPGA vs SoC vs ASIC, vendor selection, assurance level |
| **Exit review** | System Design Review (SDR) |

**Artifacts**

- System Architecture Document (**SAD**)
- HW/SW Partitioning Document
- Device selection trade study
- Interface Control Documents (**ICDs**) — system level
- Criticality classification (**DAL** for DO-254, **Category A–D** for ECSS)

---

#### ③ FPGA Requirements Specification

| | |
|---|---|
| **Purpose** | Contract between system team and FPGA team |
| **Scope** | Requirements *allocated to* the FPGA |
| **Exit review** | System Requirements Review (**SRR**) |

**Artifacts**

- Hardware Requirements Document (**HRD**) — a.k.a. FPGA Requirements Specification (**FRS**)
- Interface Requirements Specification (**IRS**) / detailed ICDs
- Requirements Traceability Matrix (**RTM**) — initialized here, lives across the whole V

---

#### ④ FPGA Architecture Design (High-Level Design — HLD)

| | |
|---|---|
| **Purpose** | Block-level decomposition and budgets |
| **Scope** | Dataflow, clocking, memory hierarchy, latency, resources |
| **Exit review** | **Preliminary Design Review (PDR)** |

**Artifacts**

| Artifact | Content |
|---|---|
| Architecture Design Document (**ADD** / HLD) | Top-level narrative, decomposition rationale |
| Block diagram & dataflow diagram | Data movement, control paths |
| Clock & reset architecture diagram | Domains, CDC strategy, reset tree |
| Memory map & memory hierarchy | Address space, URAM/BRAM/external DDR usage |
| Latency / throughput budget | Per-path numbers, margins |
| Resource budget | LUT / FF / BRAM / URAM / DSP / **AIE tile** allocation |
| Power budget | Static + dynamic per rail |
| Internal ICDs | Inter-block contracts |
| **Architecture Decision Records (ADRs)** | *Why*, not just *what*; alternatives considered |

> **Your sweet spot.** This is where the architect role earns its keep — budgets and ICDs are where architectural failures originate.

---

#### ⑤ FPGA Detailed Design (Low-Level Design — LLD / Micro-architecture)

| | |
|---|---|
| **Purpose** | Per-module internals |
| **Scope** | FSMs, pipeline stages, FIFO depths, handshakes, numerical formats |
| **Exit review** | **Critical Design Review (CDR)** |

**Artifacts**

- Detailed Design Document (**DDD**) — one master or one per significant module
- Micro-architecture diagrams (per block)
- FSM state diagrams
- Fixed-point / numerical format spec *(your DSP domain: Q-notation, saturation, rounding modes)*
- Updated **RTM** (requirements → modules)

---

### Bottom of the V — Implementation

#### ⑥ Implementation

| Category | Artifacts |
|---|---|
| **Source** | RTL (VHDL/SV) or HLS (C++ with pragmas) |
| **Constraints** | Physical + timing constraints (**XDC / SDC**) |
| **Static checks** | Lint (Spyglass, VC Lint, Questa AutoCheck), **CDC**, **RDC** reports |
| **Standards** | Coding-standard compliance (**RMM**, STARC, internal rules) |
| **Implementation reports** | Synthesis, P&R, timing, utilization, power |
| **Deliverable** | Bitstream, programming files, PDI (Versal) |
| **Configuration** | Tagged baseline in VCS |

---

### Right Side of the V — Verification & Validation

#### ⑦ Module / Unit Verification

| | |
|---|---|
| **Purpose** | Prove each block meets its spec |
| **Methodology** | UVM (complex blocks) · cocotb (Python-centric) · directed+assertion-based for simple blocks |

**Artifacts**

- Verification Plan (**VPlan**) — features, coverage model, check strategy
- Testbench + sequences
- Functional coverage model & reports
- Code coverage (line, toggle, branch, FSM, expression)
- Assertion set (**SVA** / **PSL**)
- Regression results
- Unit Verification Report

---

#### ⑧ Integration Verification

| | |
|---|---|
| **Purpose** | Prove blocks work together against the HRD |
| **Methodology** | Top-level UVM, mixed-language, HW/SW co-sim |

**Artifacts**

- Integration testbench
- System-level simulation reports (often **SystemC / MATLAB co-sim** for DSP)
- **Bit-true correlation** reports — fixed-point RTL vs floating-point reference *(core DSP artifact)*
- Integration Verification Report

---

#### ⑨ FPGA Validation (in Hardware)

| | |
|---|---|
| **Purpose** | Prove the FPGA works in its real environment |
| **Exit review** | Test Readiness Review (**TRR**) before, then validation close-out |

**Artifacts**

- **FIL** (FPGA-in-the-Loop) / **HIL** (Hardware-in-the-Loop) procedures
- Bring-up procedures
- Lab measurement reports: eye diagrams, **BER**, spectral performance, EVM
- Environmental / radiation reports: **TID**, **SEE**, thermal *(space-specific)*
- FPGA Validation Report

---

#### ⑩ System Integration & Acceptance

| | |
|---|---|
| **Purpose** | Final acceptance against mission requirements |
| **Exit review** | **Qualification Review (QR)** → **Acceptance Review (AR)** |

**Artifacts**

- Acceptance Test Procedure (**ATP**) and Results (**ATR**)
- Delivery package: bitstream + all reports + RTM closure + known-issues list

---

## 3. Cross-Cutting Artifacts (live across the entire V)

| Artifact | Purpose | When it's touched |
|---|---|---|
| **Requirements Traceability Matrix (RTM)** | Bidirectional link: mission req → FPGA req → design element → test case | Every phase |
| **Configuration Management Plan** | Baselines, VCS strategy, tagging, branching | Locked early, baselines per review |
| **Master Verification Plan** | Coverage goals, sign-off criteria | Written at PDR, updated through CDR |
| **Risk Register** | Technical & schedule risks, mitigation | Reviewed at every milestone |
| **Problem Reports (PR) / ECRs** | Issue tracking, change control | Continuous |
| **ADR log** | Architectural decisions with rationale | Whenever a non-trivial decision is made |

---

## 4. Review Gates (Milestones)

| Review | Full name | Gate meaning |
|---|---|---|
| **MCR** | Mission Concept Review | Concept is viable |
| **SRR** | System Requirements Review | Requirements are baselined |
| **SDR** | System Design Review | System architecture is sound |
| **PDR** | Preliminary Design Review | HLD + budgets are credible |
| **CDR** | Critical Design Review | LLD is ready for implementation |
| **TRR** | Test Readiness Review | Ready to run validation tests |
| **QR** | Qualification Review | Unit is qualified against spec |
| **AR** | Acceptance Review | Unit is accepted for delivery |

---

## 5. Document Hierarchy Cheat-Sheet

The three documents most often confused:

| Document | Question answered | Typical owner |
|---|---|---|
| **HRD** (Hardware Requirements Document) | *What* must the FPGA do? | Systems / Architect |
| **ADD** (Architecture Design Document) | *How* is the FPGA structured to do it? | Architect |
| **DDD** (Detailed Design Document) | *How* is each block implemented? | Module designer |

**ADRs** sit orthogonally to all three — they capture *why* a particular decision was taken, with alternatives considered. arc42 + Markdown ADRs are the pragmatic, low-overhead way to keep this log alive.

---

## 6. Acronym Glossary

| Acronym | Expansion |
|---|---|
| ADD | Architecture Design Document |
| ADR | Architecture Decision Record |
| AR | Acceptance Review |
| ATP / ATR | Acceptance Test Procedure / Report |
| BER | Bit Error Rate |
| CDC | Clock Domain Crossing |
| CDR | Critical Design Review |
| ConOps | Concept of Operations |
| DAL | Design Assurance Level (DO-254) |
| DDD | Detailed Design Document |
| ECSS | European Cooperation for Space Standardization |
| EVM | Error Vector Magnitude |
| FIL / HIL | FPGA-in-the-Loop / Hardware-in-the-Loop |
| FRS | FPGA Requirements Specification |
| HLD | High-Level Design |
| HRD | Hardware Requirements Document |
| ICD | Interface Control Document |
| IRS | Interface Requirements Specification |
| LLD | Low-Level Design |
| MCR | Mission Concept Review |
| MRD | Mission Requirements Document |
| PDR | Preliminary Design Review |
| PSL | Property Specification Language |
| QR | Qualification Review |
| RDC | Reset Domain Crossing |
| RMM | Reuse Methodology Manual |
| RTM | Requirements Traceability Matrix |
| SAD | System Architecture Document |
| SDR | System Design Review |
| SEE | Single Event Effect |
| SRR | System Requirements Review |
| SVA | SystemVerilog Assertions |
| SyRS | System Requirements Specification |
| TID | Total Ionizing Dose |
| TRR | Test Readiness Review |
| UVM | Universal Verification Methodology |
| V&V | Verification and Validation |
| VPlan | Verification Plan |
| XDC / SDC | Xilinx / Synopsys Design Constraints |
