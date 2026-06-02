# FPGA Design Parameter Estimation — DSP Modules

**BPMS-SFU FPGA Architecture Reference**  
*Version 0.2 — DSP modules only. Logic and SERDES extensions TBD.*

---

## 1. Purpose

Define a simple workflow and equation set to estimate the key FPGA design parameters for a DSP RTL module **before implementation**:

- Clock frequency range, then frozen clock value
- Input data parallelism: lane count `L`
- Output parallelism / output bus width check
- DSP58 count
- Latency
- Memory: BRAM / URAM / LUTRAM

The intent is **architectural sizing**, not exact synthesis prediction. The result should be good enough to drive clock-domain partitioning, resource budgeting, interface sizing, and early feasibility checks at SFU top level.

---

## 2. Scope

| In scope | Out of scope |
|---|---|
| Fabric DSP58 RTL modules | AIE kernels; separate framework |
| Versal ACAP, default `xcvc1802-vsva2197-1LP-i-L` | Pure-logic / control modules; extension TBD |
| Streaming and folded MAC-dominated pipelines | SERDES / Aurora line-rate interfaces; extension TBD |
| Single-rate and multirate filter chains | Mixed AIE+PL boundary sizing |
| First-order DSP, latency, and memory estimates | Exact post-synthesis utilization or timing prediction |

---

## 3. Workflow Overview

The analysis runs in three phases:

| Phase | Granularity | Output |
|---|---|---|
| **A. Per-module sizing** | Each module independently | Clock range `[f_clk_LB, f_clk_UB]`, input lane count `L`, feasible folding factors `K` |
| **B. System clock convergence** | All modules together | Minimum practical set of system clocks; frozen `f_clk` per module |
| **C. Resource estimation** | Each module at fixed `f_clk` | DSP count, latency, memory and bandwidth checks |

**Why separate B from A and C:** clock domains are a system-level decision driven by all modules collectively. Locking `f_clk` per module too early fragments the clocking architecture. Phase A produces a feasible range; Phase B chooses clocks across the SFU; Phase C estimates resources at the frozen values.

---

## 4. Notation

| Symbol | Meaning | Units |
|---|---|---|
| `R_in` | Input sample rate | Msps |
| `R_out` | Output sample rate | Msps |
| `N_MAC` | Equivalent MACs required per input sample, after normalizing any multirate or block processing to the input rate | — |
| `R_op` | Total MAC rate `= R_in · N_MAC` | MMAC/s |
| `f_clk` | Module clock frequency | MHz |
| `f_clk_UB` | Clock upper bound: planning value after closure margin | MHz |
| `f_clk_LB` | Clock lower bound: algorithm and folding driven | MHz |
| `L` | Number of input samples consumed per clock cycle | — |
| `L_out` | Number of output samples produced per clock cycle, or equivalent output bus parallelism | — |
| `M` | MAC parallelism per input lane | — |
| `K` | Folding factor `= N_MAC / M` | — |
| `η_pipe` | Pipeline efficiency / MAC duty cycle in steady state | 0–1 |
| `γ` | DSP overhead factor for extra DSP-consuming operations | ≥ 1 |
| `T_alg` | Algorithm-inherent latency | cycles or samples |
| `T_pipe` | Implementation pipeline latency | cycles |
| `T_lat` | End-to-end latency | cycles or ns |

For block-based modules, `R_in` may instead be expressed as a block/frame rate and `N_MAC` as MACs per block/frame. The equation remains:

```text
R_op = R_in · N_MAC
```

Use the same convention consistently within one module analysis. For multirate modules, an output-referenced calculation may be used as a cross-check, but the final `R_op` should match.

---

## 5. Phase A — Per-Module Sizing

### 5.1 Required inputs

For each module, collect:

1. **`R_in`, `R_out`** — fixed by the system data plan.
2. **`N_MAC` per input sample** — algorithm-dependent, normalized to input rate.
3. **`η_pipe` assumption** — expected MAC duty cycle in steady state.
4. **Allowed folding range** — algorithm dependencies that constrain how much `N_MAC` can be time-multiplexed (`K`) versus unrolled (`M`).
5. **Sample format** — real/complex and bit widths.
6. **State and memory access requirements** — coefficient memory, delay lines, scratch, reload/update ports.

Common first-order `N_MAC` cases:

| Algorithm | Equivalent `N_MAC` per input sample |
|---|---:|
| FIR direct form, `N` taps, real | `N` |
| Symmetric FIR, `N` taps, real | `N / 2` approximately, if pre-add is usable |
| Polyphase decimator by `D`, `N` taps | `N / D` |
| Polyphase interpolator by `U`, `N` taps | depends on architecture; verify with `R_op` |
| Complex multiply-accumulate | usually 4 real MACs; 3 real multipliers possible with Gauss form but with extra adds |
| FFT / filter bank | normalize block operation count to input sample rate |

For multistage chains, do **not** average the whole chain into one number unless only a rough top-level estimate is needed. Prefer sizing each stage independently, because the bottleneck stage determines local DSP usage, memory access, and timing risk.

---

### 5.2 Clock upper bound

The clock upper bound is a planning value, not the device data-sheet maximum.

```text
f_clk_UB = f_clk_raw · margin
```

where `margin` accounts for timing-closure risk, routing, utilization, control logic, memory access, and integration overhead.

For first-pass BPMS-SFU fabric DSP sizing:

| Planning class | Typical use | `f_clk_UB` |
|---|---|---:|
| Conservative | wide datapaths, BRAM/URAM access, cross-lane routing, high integration risk | 300 MHz |
| Standard | well-pipelined DSP RTL with local routing and moderate utilization | 400 MHz |
| Aggressive | localized DSP-heavy datapath with known closure evidence | 500 MHz |

**BPMS-SFU default:** use `f_clk_UB = 400 MHz` unless there is module-specific evidence that 500 MHz is realistic.

Use 500 MHz only when the design is local, deeply pipelined, and similar paths have already closed timing on the target device/speed grade. Do not size architecture to the absolute `f_clk_max`; keep at least 15–20% timing margin.

---

### 5.3 Lane count: input data parallelism

`L` is the number of input samples consumed per clock cycle.

```text
L_min = ceil(R_in / f_clk_UB)
```

Choose `L` as the smallest practical value greater than or equal to `L_min`. Power-of-2 lane counts are convenient, but not mandatory.

If `R_in <= f_clk_UB`, then `L = 1` is feasible from the input-bandwidth point of view. The module may still be compute-bound depending on `N_MAC` and folding.

If `R_in > f_clk_UB`, then `L > 1` is mandatory because the input stream cannot be accepted with one sample per cycle.

---

### 5.4 Output bandwidth check

For rate-changing modules, the output interface may require more samples per clock than the input interface.

```text
L_out_min = ceil(R_out / f_clk_UB)
```

This does **not** redefine `L`. It defines the minimum output parallelism or output bus width required to sustain the output rate.

Examples:

| Module | Usual dominant data-side constraint |
|---|---|
| Decimator | input side, because `R_in > R_out` |
| Interpolator | output side, because `R_out > R_in` |
| Single-rate module | input and output side are equal |

If `L_out_min > L`, the module needs one of the following:

- wider output bus,
- more output samples produced per cycle,
- higher `f_clk`,
- different internal architecture,
- or an explicit rate-matching stage.

---

### 5.5 Clock lower bound

For a chosen input lane count `L` and folding factor `K`:

```text
f_clk_LB(L, K) = K · R_in / (L · η_pipe)
```

where:

```text
K = N_MAC / M
```

`K = 1` means full unroll. Larger `K` means more folding and fewer MAC units per lane.

Two extremes:

| Mode | `K` | `M` per lane | `f_clk_LB` | DSP cost |
|---|---:|---:|---:|---|
| Full unroll | 1 | `N_MAC` | `R_in / (L · η_pipe)` | Maximum |
| Maximum fold | `N_MAC` | 1 | `N_MAC · R_in / (L · η_pipe)` | Minimum, often infeasible |

The feasible `K` range is the set of integer `K` values such that:

```text
f_clk_LB(L, K) <= f_clk_UB
```

Also check that `M = ceil(N_MAC / K)` is structurally meaningful. Some algorithms have legal folding factors only at specific values because of symmetry, polyphase decomposition, coefficient grouping, complex arithmetic, or memory access scheduling.

---

### 5.6 Phase A output

Per module, produce:

| Field | Value |
|---|---|
| Chosen `L` | smallest practical value ≥ `L_min` |
| Required output parallelism `L_out_min` | from output bandwidth check |
| Feasible `(K, f_clk_LB)` pairs | from §5.5 |
| Recommended operating range `[f_clk_LB, f_clk_UB]` | based on feasible `K` values |
| Candidate `K` values | record at least low-DSP and low-latency options |

Recommended `K` depends on the objective:

- Use the **highest feasible `K`** if minimizing DSP count is the priority.
- Use the **lowest feasible `K`** if minimizing latency, control complexity, and scheduling risk is the priority.
- Keep multiple options if the system clock is not frozen yet.

The output of Phase A is a **clock range and architecture option set**, not a single final value.

---

## 6. Phase B — System Clock Convergence

### 6.1 Goal

Minimize the number of distinct fabric clock domains across the SFU. Fewer clocks mean simpler CDC, less verification, fewer asynchronous FIFOs, smaller PLL/MMCM footprint, and easier timing closure.

### 6.2 Procedure

1. Tabulate all modules' `[f_clk_LB, f_clk_UB]` ranges from Phase A.
2. Identify clusters of modules whose ranges overlap.
3. For each cluster, pick a candidate system clock `f_sys` such that:

   ```text
   f_sys >= max(f_clk_LB) over all modules in the cluster
   f_sys <= min(f_clk_UB) over all modules in the cluster
   ```

4. Recheck each module at `f_sys`:
   - input lane count `L`,
   - output parallelism `L_out`,
   - feasible folding factor `K`,
   - memory ports and banking.
5. Prefer related ratios. `f_sys` values that are integer or simple-rational multiples of system reference rates are easier to generate, reason about, and verify.

### 6.3 Convergence heuristics

- Two modules with overlapping ranges should share a clock unless there is a strong reason not to.
- If a module's range does not overlap others, check whether increasing `L` can lower its required `f_clk_LB` and bring it into a shared clock cluster.
- Keep distinct fabric clocks per top-level partition to roughly **3–4 maximum**, unless the architecture clearly needs more.
- Modules at related data rates are natural candidates for a shared clock if lane counts are adjusted accordingly.
- Avoid creating a new clock domain only to save a small number of DSPs; CDC and verification cost may dominate.

### 6.4 Phase B output

| Module | Frozen `f_clk` | Frozen `L` | Required `L_out` | Derived `K` | Shared with |
|---|---:|---:|---:|---:|---|
| ... | ... | ... | ... | ... | ... |

This table is the input to Phase C.

---

## 7. Phase C — Resource Estimation at Fixed Clock

### 7.1 DSP58 count

At frozen `f_clk` and `L`, the maximum feasible folding factor is approximately:

```text
K_max = floor(f_clk · L · η_pipe / R_in)
```

Then choose a legal `K` such that:

```text
1 <= K <= K_max
```

and compute:

```text
M = ceil(N_MAC / K)
DSP_struct = L · M
DSP_total  = ceil(DSP_struct · γ)
```

where `γ` accounts for extra DSP-consuming operations not included in the main MAC count. Examples: rounding, saturation, scaling, format conversion, or auxiliary arithmetic. Default:

```text
γ = 1.2
```

Throughput sanity check:

```text
DSP_struct · f_clk · η_pipe >= R_op
```

Use typical `η_pipe` values:

| Case | `η_pipe` |
|---|---:|
| Fully unrolled streaming pipeline, no bubbles | 1.00 |
| Folded streaming design with mild control overhead | 0.90–0.95 |
| Block algorithms, heavy scheduling/control, conditional stalls | 0.70–0.85 |

Keep `γ` and `η_pipe` separate:

- `η_pipe` reduces effective throughput.
- `γ` increases estimated resource count.

---

### 7.2 Latency

Separate algorithmic latency from implementation pipeline latency:

```text
T_lat_cycles = T_alg_cycles + T_pipe_cycles + T_buffer_cycles
T_lat_seconds = T_lat_cycles / f_clk
```

The exact conversion from sample delay to clock cycles depends on lane count and where the delay is measured.

For a streaming FIR with group delay:

```text
T_group_samples = (N_taps - 1) / 2
T_group_seconds = T_group_samples / R_sample
```

If an equivalent clock-cycle estimate is needed:

```text
T_group_cycles ≈ T_group_seconds · f_clk
```

For a lane-parallel implementation, do not blindly divide group delay by `L` unless the lane ordering, commutator, and output reconstruction make that interpretation valid. The safest architectural estimate is usually in seconds first, then converted to cycles.

Implementation pipeline latency includes:

| Component | Typical contribution |
|---|---|
| DSP58 internal pipeline | usually several cycles depending on register configuration |
| Adder tree / reduction | approximately `ceil(log2 M)` stages if fully pipelined |
| Cross-lane alignment | design-specific |
| Input/output register slices | 2–4 cycles typical |
| Rate-matching FIFO | design-specific |

For early sizing, use:

```text
T_pipe_cycles = 5–15 cycles
```

unless a more specific micro-architecture is known.

---

### 7.3 Memory sizing

Memory sizing must include both **capacity** and **port bandwidth**.

| Category | Capacity estimate | Important bandwidth question | Storage choice |
|---|---|---|---|
| Coefficient ROM | `N_taps · W_coef` bits per coefficient set | How many coefficients must be read per cycle? | BRAM / URAM / distributed ROM |
| Delay line / state | roughly `N_taps · W_data · L` bits for FIR-like pipelines | How many samples must be read/written per cycle? | SRL / LUTRAM / BRAM |
| Scratch / block buffer | algorithm-specific | burst reads/writes, bank conflicts | BRAM / URAM |
| I/O FIFO | depth × width | CDC, rate matching, burst absorption | LUTRAM / BRAM |

Capacity-only estimate for BRAM18:

```text
N_BRAM18_capacity = ceil(depth / 1024) · ceil(width / 18)
```

This is only a first-order estimate. Actual BRAM use depends on primitive mode, aspect ratio, ECC, true-dual-port versus simple-dual-port mode, and synthesis packing.

Port check:

```text
reads_per_cycle  = required_read_rate  / f_clk
writes_per_cycle = required_write_rate / f_clk
```

If a memory needs more ports than the primitive provides, use one or more of:

- banking by address,
- coefficient/data replication,
- time-multiplexing at a higher internal clock,
- wider memory words,
- or a different data layout.

For high-throughput DSP modules, memory port count is often as important as raw memory capacity.

---

### 7.4 Input and output bandwidth checks

Input bandwidth:

```text
BW_in_bits = R_in · W_in
bits_per_cycle_in = BW_in_bits / f_clk
```

Output bandwidth:

```text
BW_out_bits = R_out · W_out
bits_per_cycle_out = BW_out_bits / f_clk
```

These values must match the selected stream width, lane count, and packing format.

For complex samples, define whether `W_in` and `W_out` refer to:

- one real component,
- one complex sample I+Q,
- or one packed multi-lane bus word.

Do not mix these conventions.

---

## 8. Worked Example — Polyphase Decimating FIR

### Spec

| Parameter | Value |
|---|---:|
| `R_in` | 983.04 Msps, real |
| Decimation `D` | 4 |
| `R_out` | 245.76 Msps |
| `N_taps` | 64 real coefficients |
| `W_data` | 16 bit |
| `W_coef` | 18 bit |
| `η_pipe` | 0.95 for folded options, 1.0 for fully unrolled ideal option |
| `γ` | 1.2 |

Algorithm: polyphase decimator.

Equivalent MACs per input sample:

```text
N_MAC = N_taps / D = 64 / 4 = 16
```

Total MAC rate:

```text
R_op = 983.04 · 16 = 15,728.64 MMAC/s
```

---

### Phase A

Use standard BPMS-SFU planning value:

```text
f_clk_UB = 400 MHz
```

Input lane count:

```text
L_min = ceil(983.04 / 400) = 3
```

Choose:

```text
L = 4
```

Output bandwidth check:

```text
L_out_min = ceil(245.76 / 400) = 1
```

So the input side dominates.

Clock lower-bound candidates at `L = 4`:

| `K` | `M = ceil(N_MAC/K)` per lane | `η_pipe` | `f_clk_LB = K·R_in/(L·η_pipe)` | DSP total estimate |
|---:|---:|---:|---:|---:|
| 1 | 16 | 1.00 | 245.76 MHz | `ceil(4·16·1.2)` = 77 |
| 2 | 8 | 0.95 | 517.39 MHz | infeasible > 400 MHz |
| 4 | 4 | 0.95 | 1034.78 MHz | infeasible > 400 MHz |

With `f_clk_UB = 400 MHz`, only the full-unroll option is feasible at `L = 4`.

If DSP count is too high, increase `L` and test again. For example, `L = 8`:

| `K` | `M` per lane | `η_pipe` | `f_clk_LB` | DSP total estimate |
|---:|---:|---:|---:|---:|
| 1 | 16 | 1.00 | 122.88 MHz | 154 |
| 2 | 8 | 0.95 | 258.69 MHz | 77 |
| 4 | 4 | 0.95 | 517.39 MHz | infeasible > 400 MHz |

`L = 8, K = 2` gives the same approximate DSP count as `L = 4, K = 1`, but uses a wider datapath and lower clock. It may be useful if system clock convergence favors a lower clock, but it is usually not preferable unless there is a clear system-level reason.

Phase A candidate options:

| Option | `L` | `K` | `f_clk_LB` | `f_clk_UB` | DSP estimate | Comment |
|---|---:|---:|---:|---:|---:|---|
| A | 4 | 1 | 245.76 | 400 | 77 | Standard option |
| B | 8 | 2 | 258.69 | 400 | 77 | Wider datapath, similar DSP, lower-clock clustering option |
| C | 8 | 1 | 122.88 | 400 | 154 | Lower clock, higher DSP; likely unattractive |

---

### Phase B — illustrative

Suppose the SFU has:

| Module | `f_clk_LB` | `f_clk_UB` |
|---|---:|---:|
| M1, hypothetical CFR | 300 | 400 |
| M2, this FIR Option A | 245.76 | 400 |
| M3, low-rate control DSP | 200 | 300 |

A reasonable convergence is:

| Cluster | Modules | Frozen `f_clk` |
|---|---|---:|
| DSP high-rate cluster | M1, M2 | 400 MHz |
| Low-rate cluster | M3 | 250 MHz |

Frozen for this module:

```text
f_clk = 400 MHz
L = 4
K = 1
```

---

### Phase C at `f_clk = 400 MHz, L = 4, K = 1`

| Quantity | Calculation | Value |
|---|---|---:|
| `M` per lane | `ceil(16 / 1)` | 16 |
| `DSP_struct` | `4 · 16` | 64 |
| `DSP_total` | `ceil(64 · 1.2)` | 77 |
| Throughput check | `64 · 400 · 1.0` vs `15,728.64` | 25,600 ≥ 15,728.64 ✓ |
| Output parallelism | `ceil(245.76 / 400)` | 1 sample/cycle minimum |

Latency estimate:

```text
T_group_samples = (64 - 1) / 2 = 31.5 input samples
T_group_seconds = 31.5 / 983.04e6 = 32.0 ns
```

Assume implementation pipeline:

```text
T_pipe_cycles = 10 cycles at 400 MHz = 25 ns
```

Total rough latency:

```text
T_lat ≈ 32 ns + 25 ns = 57 ns
```

Memory estimate:

| Memory | First-order capacity | Comment |
|---|---:|---|
| Coefficient ROM | `64 · 18 = 1152 bit`, replicated as needed for parallel coefficient reads | capacity small; port replication may dominate |
| Delay/state | approximately `64 · 16 · 4 = 4096 bit` | LUTRAM/SRL or BRAM depending architecture |
| I/O FIFO | e.g. 32 entries × bus width | driven by CDC/rate matching, not FIR math |

The coefficient memory capacity is trivial, but the design may need multiple coefficient reads per cycle. Do not declare memory complete until coefficient-port scheduling is checked.

---

## 9. Per-Module Analysis Template

Copy this block per module.

```text
Module name:        ____________________
Block diagram ref:  ____________________
Algorithm:          ____________________

INPUTS
R_in  [Msps]:       ____
R_out [Msps]:       ____
N_MAC per input:    ____
R_op  [MMAC/s]:     ____
W_data / W_coef:    ____ / ____ bit
η_pipe assumption:  ____
γ assumption:       ____

PHASE A — Per-module sizing
f_clk_UB [MHz]:     ____   (default 400; 500 only with justification)
L_min:              ____
Chosen L:           ____
L_out_min:          ____
f_clk_LB at K=1:    ____ MHz
f_clk_LB at K=2:    ____ MHz
Feasible K values:  ____
Feasible range:     [____ , ____] MHz
Candidate options:  ____

PHASE B — After system convergence
Frozen f_clk [MHz]: ____
Frozen L:           ____
Required L_out:     ____
Derived K:          ____
Shared clock with:  ____________________

PHASE C — Resource estimation
M (per lane):       ____
DSP_struct:         ____
DSP_total (γ=1.2):  ____
Throughput check:   DSP·f·η = ____ MMAC/s vs R_op = ____ ✓/✗
Input bits/cycle:   ____
Output bits/cycle:  ____
T_alg:              ____ samples / ____ ns
T_pipe:             ____ cycles / ____ ns
T_lat:              ____ ns
Coefficient ROM:    ____ BRAM18 / ____ URAM / ____ LUTRAM
Delay line/state:   ____ BRAM18 / ____ URAM / ____ LUTRAM
I/O FIFO:           ____ BRAM18 / ____ LUTRAM
Memory port check:  ____ reads/cycle, ____ writes/cycle ✓/✗

NOTES / RISKS:
________________________________________
________________________________________
```

---

## 10. Default Values — BPMS-SFU First-Pass Planning

| Parameter | Default | Comment |
|---|---:|---|
| Device | `xcvc1802-vsva2197-1LP-i-L` | update if target changes |
| Dev platform | VCK190 / VC1902 | for early bring-up only; production device may differ |
| `f_clk_UB`, conservative | 300 MHz | wide datapaths, routing risk, BRAM/URAM-heavy |
| `f_clk_UB`, standard | 400 MHz | default for fabric DSP sizing |
| `f_clk_UB`, aggressive | 500 MHz | only with closure evidence |
| `η_pipe`, fully streaming | 1.00 | no bubbles, full unroll |
| `η_pipe`, folded streaming | 0.90–0.95 | mild scheduling/control overhead |
| `η_pipe`, block/control-heavy | 0.70–0.85 | FFT/matrix/block algorithms |
| `γ` | 1.20 | DSP overhead for aux arithmetic |
| Closure margin | 15–20% below demonstrated `fmax` | do not size to raw maximum |
| Max distinct fabric clocks per top | 3–4 | CDC and verification budget |

Update these values when measured implementation data exists.

---

## 11. Common Pitfalls

- **Sizing to raw `f_clk_max` instead of `f_clk_UB`** — leaves no closure margin.
- **Using 500 MHz as default without evidence** — may under-estimate lanes and memory banking.
- **Forgetting `η_pipe` in folding feasibility** — makes folded options look feasible when they are not.
- **Confusing `γ` and `η_pipe`** — `γ` increases resources; `η_pipe` reduces effective throughput.
- **Forgetting output bandwidth** — interpolators and expanding blocks can be output-limited even when input `L` is small.
- **Counting `N_MAC` at the wrong rate** — for multirate filters, input-referenced and output-referenced calculations must give the same `R_op`.
- **Ignoring complex-vs-real arithmetic** — complex MACs are not one real MAC.
- **Assuming all folding factors are legal** — memory access, symmetry, polyphase structure, and coefficient scheduling may restrict `K`.
- **Capacity-only memory estimates** — BRAM/URAM port count can dominate over raw bit count.
- **Ignoring coefficient reload paths** — adaptive filters or live retuning need update ports and coherency rules.
- **Blindly converting sample latency to lane-divided cycles** — group delay is safest estimated in seconds first.
- **Not retesting `L` and `K` after Phase B** — a lower shared clock may invalidate Phase A assumptions.

---

## 12. Extensions Planned

| Section | Scope |
|---|---|
| Logic-only modules | Replace `N_MAC` with operations/event; bottleneck shifts to LUT depth, routing, memory ports, state dependencies |
| SERDES / Aurora | Line rate, gearbox ratios, encoding overhead, RX elastic buffer, lane alignment latency |
| AIE companion sizing | AIE v1 tile count, vector ops/cycle, memory banks, AIE-PL boundary throughput |
| Mixed AIE + RTL | Cross-domain throughput matching, AXI-Stream sizing at PL↔AIE boundary |
| Power estimation | activity factors, toggle rate, clock tree, memory/DSP utilization |

---

## 13. Revision Log

| Version | Date | Author | Changes |
|---|---|---|---|
| 0.1 | 2026-05-06 | Marco | Initial DSP-only draft |
| 0.2 | 2026-05-06 | ChatGPT | Incorporated review updates: conservative clock planning, input-lane/output-bandwidth split, `η_pipe` in folding, refined DSP/memory/latency checks, updated worked example |
