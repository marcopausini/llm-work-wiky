# Filter-Bank Throughput Derivation — BPMS/SFU Learning Notes

## 1. Purpose

This note summarizes the reasoning developed in the chat to understand how a **filter bank** fits into the BPMS signal chain and how to derive the first-order **throughput, clock, lane, and DSP resource implications** for the SFU downlink filter bank.

The focus is educational: start from system-level quantities, derive sample rates and frame rates, then connect those numbers to FPGA architecture decisions.

---

## 2. What is a filter bank?

A **filter bank** is a DSP structure that converts between:

- a **wideband time-domain signal**, and
- many **narrow frequency-domain bins/subbands**.

There are two complementary operations:

```text
Analysis filter bank:
  time-domain waveform → frequency-domain bins

Synthesis filter bank:
  frequency-domain bins → time-domain waveform
```

In BPMS, this bin-based representation is central. It allows the system to represent LTE/5G beams as spectral-bin allocations, move them through the DCU/SFU interconnect, and reconstruct the final Q/V-band RF waveform at the SFU.

---

## 3. Why BPMS uses filter banks

Without a filter bank, each beam would remain an independent time-domain stream, and the system would need to combine many separate LTE/5G waveforms directly in time domain. That is inefficient and difficult to route.

With a filter bank, each beam is converted into frequency bins. Those bins can then be packed, routed, selected, and reconstructed.

The practical benefits are:

1. **Frequency-domain packing**  
   Each AxC / beam occupies a fixed number of contiguous spectral bins.

2. **Mixed bandwidth support**  
   Different LTE bandwidths can coexist in the same OBG link because everything is represented as bin ranges.

3. **Efficient FPGA implementation**  
   Shared polyphase/FFT structures are more efficient than one independent DUC/DDC chain per beam.

4. **Clean DCU/SFU ownership split**  
   The DCU handles beam-level functions. The SFU handles band/sub-band-level functions.

---

## 4. DCU vs SFU filter-bank roles

The DCU and SFU both use filter-bank concepts, but their roles are different.

| Device | Direction | Filter-bank role |
|---|---|---|
| DCU | DL | Convert time-domain AxC/beam samples into spectral bins |
| SFU | DL | Convert selected spectral bins into wideband time-domain RF sub-band |
| SFU | UL | Convert wideband time-domain RF sub-band into spectral bins |
| DCU | UL | Convert returned spectral bins back toward beam/CPRI representation |

For downlink:

```text
eNodeB CPRI
  → DCU: time-domain beams → spectral bins
  → OBG links / optical matrix
  → SFU: spectral bins → Q/V RF waveform
```

Important architectural boundary:

```text
DCU owns:
  per-beam gain
  per-beam Doppler
  per-beam delay compensation
  satellite bin-grid generation/alignment

SFU owns:
  OBG ingest
  bin selection/routing
  filter-bank synthesis/analysis
  band gain
  band Doppler
  RF interfacing
```

Do not move per-beam processing into the SFU unless the system architecture explicitly changes.

---

## 5. First interface: eNodeB to DCU for LTE 10 MHz

Assumption used in the chat:

```text
LTE 10 MHz beam sample rate over CPRI = 15.36 Msps complex
sample format = 16-bit I + 16-bit Q = 32 bits / complex sample
```

For 10 MHz LTE:

```text
1 eNodeB = 32 beams
1 eNodeB uses 2 CPRI lanes
1 DCU has 12 CPRI inputs
```

Therefore:

```text
1 DCU receives 6 eNodeBs
6 eNodeBs × 32 beams/eNodeB = 192 beams
```

Per beam payload rate:

```text
15.36e6 samples/s × 32 bits/sample
= 491.52 Mb/s
```

Per eNodeB:

```text
32 beams × 491.52 Mb/s
= 15.72864 Gb/s
```

Per DCU:

```text
192 beams × 491.52 Mb/s
= 94.37184 Gb/s
```

These are raw IQ payload rates, not necessarily full CPRI line rates including framing and protocol overhead.

---

## 6. DCU beam resampling and bin generation

For FPGA satellite operation, the beam is assumed to be resampled to:

```text
10.24 Msps complex per 10 MHz beam
```

Payload rate per beam after SRC:

```text
10.24e6 samples/s × 32 bits/sample
= 327.68 Mb/s
```

For 192 beams:

```text
192 × 10.24e6 = 1.96608 Gcomplex samples/s
192 × 327.68 Mb/s = 62.91456 Gb/s
```

For FPGA satellite, a 10.24 MHz beam maps to:

```text
120 bins
```

Bin spacing:

```text
10.24 MHz / 120 = 85.333 kHz
```

For 192 beams:

```text
192 beams × 120 bins/beam = 23,040 bins
```

A DCU has 4 OBG outputs, and each OBG link carries 5,760 bins:

```text
4 × 5,760 bins = 23,040 bins
```

So the all-10 MHz / FPGA-satellite example fits exactly:

```text
192 beams × 120 bins/beam
= 23,040 bins
= 4 full OBG links
```

Equivalent view:

```text
1 OBG link = 5,760 bins
5,760 bins / 120 bins per beam = 48 beams per OBG link
```

---

## 7. Clarifying the “samples/s/bin” point

A key correction from the chat:

```text
85.333 kHz is bin spacing / frequency resolution.
```

It is **not automatically** a sample rate.

However, in a critically sampled uniform filter bank, each bin stream may be interpreted as updating at:

```text
R_bin = Fs / N_bins
```

For one 10.24 Msps beam:

```text
R_bin = 10.24e6 / 120
      = 85.333 ksample/s per bin stream
```

The numerical value equals the bin spacing, but the physical meaning is different:

| Quantity | Meaning |
|---|---|
| 85.333 kHz | frequency spacing / bin bandwidth |
| 85.333 ksample/s/bin | inferred bin-stream update rate under a critically sampled model |

For clean architecture documentation, avoid writing the bin-stream rate as a hard fact unless the OBG/frame protocol confirms it. Prefer:

```text
[INFERRED] Assuming critically sampled bin streams:
  R_bin = Fs / N_bins
```

The unambiguous payload-rate computation is:

```text
192 beams × 10.24 Msps/beam × 32 bits/sample
= 62.91456 Gb/s
```

---

## 8. General FPGA DSP throughput methodology

For any DSP module, start from throughput before thinking about implementation.

Recommended sequence:

```text
1. Input sample rate / frame rate
2. Output sample rate / frame rate
3. Input/output payload width
4. Required samples per second
5. Candidate clock frequency
6. Required samples per clock
7. Algorithmic operations per sample or per frame
8. Required operations per second
9. Required operations per cycle
10. DSP48 / memory / lane estimate
```

Basic equations:

```text
input_payload_rate  = R_in  × W_in
output_payload_rate = R_out × W_out
```

```text
samples_per_cycle_required = sample_rate / f_clk
```

```text
required_lanes = ceil(sample_rate / f_clk)
```

For arithmetic:

```text
operations_per_second = reference_rate × operations_per_sample
```

or for frame-based blocks:

```text
operations_per_second = frame_rate × operations_per_frame
```

DSP estimate:

```text
N_dsp ≥ operations_per_second
        / (f_clk × useful_operations_per_cycle_per_dsp × efficiency)
```

Efficiency must be less than 1.0 because real FPGA designs have pipeline bubbles, frame-boundary overhead, memory banking limits, routing pressure, and backpressure effects.

A practical early estimate may use:

```text
efficiency ≈ 0.6 to 0.85
```

depending on how regular the computation is.

---

## 9. Important architecture principle: clock first, then lanes

The FPGA clock should not be independently invented per module.

Better approach:

```text
Define a small set of architecture-wide clock domains.
Then size each module's parallelism to meet throughput in those domains.
```

For each module, the Module Spec should state:

```text
required input throughput
required output throughput
clock domain
minimum sustained samples/cycle
allowed backpressure behavior
latency budget
frame size
boundary fixed-point format
overflow/saturation behavior
telemetry/status requirements
```

The Module Spec should generally **not** prescribe:

```text
exact DSP48 count
exact pipeline structure
exact FIFO depth
exact internal lane scheduling
exact memory banking
```

Those belong to the RTL designer’s micro-architecture unless they are hard system constraints.

---

## 10. Applying the method to the SFU DL filter bank from scratch

Now apply the clean-sheet method to the SFU **DL filter-bank synthesis** block.

We deliberately ignore the existing implementation clocks and lane counts.

Known requirements:

```text
One filter-bank synthesis instance per sub-band
Input: spectral bins for one sub-band
Output: time-domain complex waveform for one sub-band

N_bins_per_frame = 12,288 bins
F_out            = 1,024 Msps complex
```

The DL SFU filter bank is synthesis:

```text
frequency-domain bins → time-domain sub-band waveform
```

---

## 11. Deriving frame rate and bin spacing

For a critically sampled synthesis filter bank, start with:

```text
N_out_samples_per_frame = N_bins_per_frame
```

Therefore:

```text
N_out = 12,288 complex samples/frame
```

Frame rate:

```text
frame_rate = F_out / N_out
           = 1,024e6 / 12,288
           = 83,333.333 frames/s
```

Frame period:

```text
T_frame = 1 / 83,333.333
        = 12 µs
```

Bin spacing:

```text
Δf = F_out / N_bins
   = 1,024e6 / 12,288
   = 83.333 kHz
```

So the first-principles interpretation is:

```text
12,288 bins across 1,024 MHz
→ 83.333 kHz/bin
→ 12 µs frame period
```

---

## 12. Input and output throughput for one SFU DL sub-band

Each input spectral frame contains:

```text
12,288 complex bin values
```

The block must process:

```text
83,333.333 frames/s
```

So the input throughput is:

```text
R_in = 12,288 bins/frame × 83,333.333 frames/s
     = 1.024e9 complex bin-samples/s
```

Output throughput is given:

```text
R_out = 1.024e9 complex samples/s
```

Thus for a critically sampled synthesis bank:

```text
R_in = R_out = 1.024 Gcomplex samples/s per sub-band
```

The block changes representation, not the total complex-sample throughput.

---

## 13. Payload-rate examples

Payload rate depends on boundary fixed-point width.

If the filter-bank boundary is 16-bit I + 16-bit Q:

```text
W = 32 bits/complex sample

payload_rate = 1.024e9 × 32
             = 32.768 Gb/s per sub-band
```

If the boundary is 18-bit I + 18-bit Q:

```text
W = 36 bits/complex sample

payload_rate = 1.024e9 × 36
             = 36.864 Gb/s per sub-band
```

So the Module Spec must explicitly define the boundary data width. Without that, only the complex-sample throughput is known.

---

## 14. Choosing clock and lanes

Once throughput is known, choose a candidate clock and derive required parallelism.

General rule:

```text
P × f_clk ≥ 1.024e9 complex samples/s
```

where:

```text
P = number of complex samples processed per cycle
```

Equivalently:

```text
P = ceil(1.024e9 / f_clk)
```

Examples:

| Candidate clock | Required samples/cycle | Minimum lanes |
|---:|---:|---:|
| 128 MHz | 8.000 | 8 |
| 160 MHz | 6.400 | 7 |
| 204.8 MHz | 5.000 | 5 |
| 256 MHz | 4.000 | 4 |
| 320 MHz | 3.200 | 4 |
| 341.333 MHz | 3.000 | 3 |
| 409.6 MHz | 2.500 | 3 |
| 512 MHz | 2.000 | 2 |

This is the basic clock/parallelism trade:

```text
higher clock → fewer lanes
lower clock  → more lanes
```

But higher clock is not always better. Very high fabric clocks increase timing-closure risk, routing pressure, and power.

---

## 15. Frame-level lane equation

The same result can be derived from frames.

For `P` lanes:

```text
cycles_per_frame = N_bins / P
```

The clock must satisfy:

```text
f_clk ≥ cycles_per_frame × frame_rate
```

Equivalent:

```text
f_clk ≥ (N_bins / P) × frame_rate
```

Example with 4 lanes:

```text
cycles_per_frame = 12,288 / 4
                 = 3,072 cycles

f_clk ≥ 3,072 × 83,333.333
      = 256 MHz
```

Example with 3 lanes:

```text
cycles_per_frame = 12,288 / 3
                 = 4,096 cycles

f_clk ≥ 4,096 × 83,333.333
      = 341.333 MHz
```

So a `3 lanes @ 341.333 MHz` implementation is not arbitrary. It follows from:

```text
12,288 bins/frame
83.333 kframes/s
3 samples/cycle
```

But when specifying from scratch, do not start there. Start from throughput, then evaluate architectural options.

---

## 16. Clean Module Spec wording for the SFU DL filter bank

At MS level, before locking implementation lanes/clocks, the requirement can be written as:

```text
The DL synthesis filter bank shall process one SFU sub-band.

For ASIC-satellite mode:
  output sample rate: 1,024 Msps complex
  input spectral frame size: 12,288 complex bins/frame
  derived frame rate: 83.333333 kframes/s
  derived bin spacing: 83.333333 kHz
  sustained input throughput: 1.024 Gcomplex bin-samples/s
  sustained output throughput: 1.024 Gcomplex samples/s

The implementation shall sustain continuous operation with no frame loss.

The implementation shall choose clock frequency and parallelism such that:
  f_clk × complex_samples_per_cycle ≥ 1.024e9
```

Then the FAD or MS can list candidate implementation operating points:

```text
Option A: 4 lanes @ 256 MHz
Option B: 3 lanes @ 341.333 MHz
Option C: 2 lanes @ 512 MHz
```

The selected option should consider timing closure, resource usage, vendor IP constraints, RFdc clock relationships, and integration complexity.

---

## 17. Arithmetic-load estimate comes after throughput

For the MDFT synthesis filter bank, arithmetic cost is not determined by throughput alone.

The compute cost per frame includes:

```text
polyphase FIR / pre-post filtering
FFT or inverse-modulation stages
scaling
rounding
saturation / clipping detection
reorder / framing logic
```

First-order equation:

```text
compute_per_second = compute_per_frame × frame_rate
```

Then:

```text
N_dsp ≥ compute_per_second
        / (f_clk × useful_ops_per_cycle_per_dsp × efficiency)
```

This step requires algorithm details: number of taps, FFT structure, radix decomposition, coefficient symmetry, complex multiplier strategy, and internal fixed-point policy.

So the correct design order is:

```text
signal throughput first
clock/lane trade second
algorithmic MAC/frame estimate third
DSP48/resource estimate fourth
```

---

## 18. Summary

For one SFU DL synthesis filter bank, starting from scratch:

```text
Given:
  N_bins = 12,288 bins/sub-band frame
  F_out  = 1,024 Msps complex

Derived:
  frame_rate = 83.333333 kframes/s
  frame_period = 12 µs
  bin_spacing = 83.333333 kHz
  input throughput = 1.024 Gcomplex bin-samples/s
  output throughput = 1.024 Gcomplex samples/s
```

Implementation sizing condition:

```text
P × f_clk ≥ 1.024e9 complex samples/s
```

where:

```text
P = complex samples per cycle
```

This is the clean architectural starting point. From there, choose a small set of global SFU clock domains, size the number of lanes, and only then estimate DSP48 resources from the actual MDFT/polyphase algorithm.
