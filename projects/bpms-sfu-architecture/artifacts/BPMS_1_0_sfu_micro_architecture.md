# BPMS 1.0 — SFU Block-Level Micro-Architecture

**Document:** Initial SFU FPGA micro-architecture decomposition  
**Primary source:** BPMS-1.0-SFU-001 v1.6  
**Context source:** BPMS-1.0-ARCH-001 v2.4  
**Date:** April 2026  
**Author role:** FPGA architect — SFU micro-architecture  

**Convention:** [SPEC] = stated in SFU doc. [CTX] = from system doc only. [INF] = inferred by reviewer. [PROP] = design proposal. [TBD] = explicitly open in the documents.

---

## 1. Top-Level Block Decomposition

The SFU FPGA design is decomposed into the following top-level blocks. Each block maps to one or more identifiable processing stages in the SFU document.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  RFSoC PL (Programmable Logic)                                              │
│                                                                             │
│  ┌──────────────┐   ┌───────────┐   ┌───────────────┐   ┌──────────────┐  │
│  │ aurora_rx_tx  │──▶│ dl_cdc    │──▶│ lane_align    │──▶│ bins_sel_dl  │  │
│  │ (4-lane)      │   │ (×4 lane) │   │ (4-lane)      │   │              │  │
│  │               │◀──│ ul_cdc    │◀──────────────────────│ bins_sel_ul  │  │
│  └──────┬───────┘   └───────────┘   └───────────────┘   └──────┬───────┘  │
│         │                                                       │          │
│         │  ┌──────────────────────────────────────────────────┐  │          │
│         │  │              fb_core (×2 sub-band)               │  │          │
│         │  │  ┌────────────┐              ┌────────────────┐  │  │          │
│         │  │  │ fb_synth   │◀─────────────│ bins_sel_dl    │──┘  │          │
│         │  │  │ (DL)       │              └────────────────┘     │          │
│         │  │  └─────┬──────┘                                     │          │
│         │  │        ▼                                            │          │
│         │  │  ┌────────────┐                                     │          │
│         │  │  │ beacon_inj │  (DL only, before band_gain)        │          │
│         │  │  └─────┬──────┘                                     │          │
│         │  │        ▼                                            │          │
│         │  │  ┌────────────┐                                     │          │
│         │  │  │ band_gain  │  (DL and UL instances)              │          │
│         │  │  └─────┬──────┘                                     │          │
│         │  │        ▼                                            │          │
│         │  │  ┌────────────┐                                     │          │
│         │  │  │ band_dop   │  (DL and UL instances)              │          │
│         │  │  └─────┬──────┘                                     │          │
│         │  │        ▼                                            │          │
│         │  │  ┌────────────────┐    ┌────────────────┐           │          │
│         │  │  │ fb_analysis    │───▶│ bins_sel_ul    │───────────┘          │
│         │  │  │ (UL)           │    └────────────────┘                      │
│         │  │  └────────────────┘                                            │
│         │  └────────────────────────────────────────────────────┘           │
│         │                                                                   │
│  ┌──────┴───────┐   ┌───────────────┐   ┌───────────────┐                  │
│  │ rf_port_sel   │──▶│ duc_ddc       │──▶│ adc_dac_if    │ ◀──▶ RF ports   │
│  └───────────────┘   └───────────────┘   └───────────────┘                  │
│                                                                             │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐                  │
│  │ timing_core   │   │ param_engine  │   │ mgmt_regs     │ ◀──▶ AXI (PS)   │
│  │ (1pps, clk)   │   │ (utc_sched)   │   │               │                  │
│  └───────────────┘   └───────────────┘   └───────────────┘                  │
│                                                                             │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐                  │
│  │ playback      │   │ capture       │   │ spur_det      │                  │
│  └───────────────┘   └───────────────┘   └───────────────┘                  │
│                                                                             │
│  ┌───────────────┐                                                          │
│  │ latency_meas  │                                                          │
│  └───────────────┘                                                          │
│                                                                             │
│  RFSoC PS (ARM Cortex-A53)                                                  │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐                  │
│  │ tle_doppler   │   │ mgmt_sw       │   │ nv_config     │                  │
│  │ (SGP4 compute)│   │ (1GbE agent)  │   │ (flash I/O)   │                  │
│  └───────────────┘   └───────────────┘   └───────────────┘                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Block-by-Block Specification

### 2.1 aurora_rx_tx

**Source:** SFU §6.1, §7.5, §10.3, §11.1, §11.2

| Attribute | Value | Origin |
|-----------|-------|--------|
| Protocol | Xilinx Aurora 64B/66B (PG074) | [SPEC] ref [5] |
| Lanes | 4, full-duplex via Q0 QSFP28 | [SPEC] §6.1 |
| Line rate | 19.2 Gbps per lane | [SPEC] §10.3 |
| Breakout | 1×QSFP28 → 4×LC duplex external cable | [SPEC] §10.3 |
| CDR timing master | One pre-selected DL lane per SFU; configured at commissioning (UTC-Scheduled) | [SPEC] §6.1, §11.1 |
| Clock output (primary) | Recovered clock from CDR → drives all PL logic + ADC/DAC | [SPEC] §11.1 |
| Clock output (secondary) | UL lanes driven by 10 MHz-derived clock from CI port PLL | [SPEC] §11.2 |
| Frame format | **Not defined** | [TBD] — no ICD in either document |

**Interfaces:**

| Port | Direction | Width / type | Destination |
|------|-----------|-------------|-------------|
| dl_data[0:3] | out | OBG frame bus (IQ bins) | dl_cdc[0:3] |
| ul_data[0:3] | in | OBG frame bus (IQ bins) | from ul_cdc[0:3] |
| aurora_clk | out | clock | Recovered CDR clock (primary mode source) |
| lane_status[0:3] | out | 1-bit per lane | mgmt_regs (Up/Down, alarm on loss) [SPEC] §14.2 |
| cdr_lock | out | 1-bit | mgmt_regs (Locked/Unlocked, alarm on unlock) [SPEC] §14.2 |

**Open questions:**

| # | Question | Impact |
|---|----------|--------|
| AUR-1 | OBG frame format: header fields, bin ordering (frequency-ascending? interleaved?), metadata, CRC/ECC, idle pattern | Blocks Rx/Tx framer and Bins Selector RTL |
| AUR-2 | Aurora flow control: is back-pressure supported, or is the link free-running at line rate? | Affects CDC FIFO sizing and overflow handling |
| AUR-3 | Aurora initialization sequence and lane bonding requirements for the 4-lane configuration | Affects bring-up firmware |

---

### 2.2 dl_cdc / ul_cdc

**Source:** SFU §6.1 (DL CDC), §7.5 (UL implicit)

| Attribute | Value | Origin |
|-----------|-------|--------|
| Purpose | Bridge Aurora SERDES clock domain ↔ DSP processing clock domain | [SPEC] §6.1 |
| Instances | 4× DL (one per Aurora Rx lane), 4× UL (one per Aurora Tx lane) | [INF] from 4-lane architecture |
| DL source clock | Aurora CDR-recovered clock (per-lane) | [SPEC] §6.1 |
| DL destination clock | DSP processing clock (341.333 MHz at FB input) | [SPEC] §6.4 |
| UL source clock | DSP processing clock (341.333 MHz at FB output) | [SPEC] §7.4 |
| UL destination clock | Aurora Tx clock (CDR-derived primary, or 10 MHz-derived secondary) | [INF] |
| Correction events | Atomic, at frame boundaries only, timestamped, event-logged, reported to Manager SW | [SPEC] §6.1 |
| Pipeline latency register | Updated immediately on any CDC correction event | [SPEC] §6.1 |

**Interfaces:**

| Port | Direction | Width / type | Notes |
|------|-----------|-------------|-------|
| data_in | in | OBG frame bus | From aurora_rx_tx (DL) or bins_sel_ul (UL) |
| data_out | out | OBG frame bus | To lane_align (DL) or aurora_rx_tx (UL) |
| src_clk | in | clock | Source clock domain |
| dst_clk | in | clock | Destination clock domain |
| correction_event | out | pulse + metadata | To latency_meas, mgmt_regs event log |
| fifo_level | out | pointer diff | To mgmt_regs telemetry |

**Design decisions needed:**

| # | Decision | Options |
|---|----------|---------|
| CDC-1 | Correction mechanism | [PROP] Pointer adjustment (drop or repeat one frame). Sample insert/delete is more complex and less predictable. Frame-boundary-only constraint [SPEC] supports pointer adjustment. |
| CDC-2 | FIFO depth | [INF] Must accommodate worst-case clock frequency offset between Aurora CDR and DSP clock. With both derived from the same GPS-traceable source, offset is sub-ppm → shallow FIFO (16–32 frames). Secondary mode (independent 10 MHz) may have larger offset → deeper FIFO or tighter PLL spec. |
| CDC-3 | Per-lane or shared FIFO? | [PROP] Per-lane. Each Aurora lane has its own CDR; phase alignment happens downstream in lane_align. |

---

### 2.3 lane_align

**Source:** SFU §6.2

| Attribute | Value | Origin |
|-----------|-------|--------|
| Purpose | Compensate inter-lane phase skew across 4 OBG DL lanes before Bins Selector | [SPEC] §6.2 |
| Skew sources | Differential optical fibre length through Polatis matrix + differential PCB routing | [SPEC] §6.2 |
| Mechanism | Per-lane programmable delay register | [SPEC] §6.2 |
| Reference | Pre-selected timing master OBG lane (same as CDR source) | [SPEC] §6.2 |
| Configuration | UTC-Scheduled, set at commissioning, stored NV | [SPEC] §6.2 |
| Max pre-compensation skew | TBD (OI-S02) | [TBD] |
| Scope | Intra-SFU only (distinct from inter-SFU coherence §11.5) | [SPEC] §6.2 |

**Interfaces:**

| Port | Direction | Width / type | Notes |
|------|-----------|-------------|-------|
| lane_in[0:3] | in | OBG frame bus (post-CDC) | From dl_cdc[0:3] |
| lane_out[0:3] | out | OBG frame bus (aligned) | To bins_sel_dl |
| delay_reg[0:3] | in | config word | From param_engine (UTC-Scheduled) |
| master_lane_sel | in | 2-bit | Selects reference lane |

**Design proposal:**

[PROP] Implement as per-lane BRAM-based circular buffer with programmable read pointer offset relative to write pointer. The reference lane has zero offset; other lanes are delayed to match. Depth sized for max expected skew. Pending OI-S02; conservative estimate: 100 ns → ~34 samples at 341.333 MHz → 64-deep buffer per lane is safe with margin.

[PROP] Alignment verification: expose per-lane delay measurement via mgmt_regs — compare frame boundary timestamps across lanes to confirm alignment within tolerance.

---

### 2.4 bins_sel_dl

**Source:** SFU §6.3

| Attribute | Value | Origin |
|-----------|-------|--------|
| Purpose | Select bins per AxC from 4 OBG lanes; route to correct FB sub-band at correct spectral position | [SPEC] §6.3 |
| Input | Full bin payload from all 4 aligned OBG lanes | [SPEC] §6.3 |
| Output | Bin streams routed to fb_synth sub-band A and sub-band B at configured spectral positions | [SPEC] §6.3 |
| BW-agnostic | Yes — maps bin ranges regardless of cell BW | [SPEC] §6.3 |
| Configuration per AxC | {OBG lane, bin offset, bin count} → {sub-band A or B, bin slot index} | [SPEC] §6.3 |
| Mixed-BW | Supported within same OBG lane and same sub-band | [SPEC] §6.3, SFU-SIG-05 |
| Max input bins | 4 × 5,760 = 23,040 | [INF] from 4 lanes × OBG capacity |
| Max output bins/sub-band | 12,288 | [SPEC] §6.4 |
| Max occupied RF bins/SFU | 19,200 | [SPEC] §12.1 |
| Configuration apply | UTC-Scheduled | [SPEC] §6.3 |
| FPGA satellite | No modification needed — DCU provides natively aligned bins | [SPEC] §6.3 |

**Interfaces:**

| Port | Direction | Width / type | Notes |
|------|-----------|-------------|-------|
| lane_bins[0:3] | in | Bin stream (aligned, from lane_align) | Up to 5,760 bins/lane |
| sb_a_bins | out | 12,288-bin frame to fb_synth sub-band A | Bin position determines RF frequency |
| sb_b_bins | out | 12,288-bin frame to fb_synth sub-band B | |
| config_table | in | NMS-configured mapping table | From param_engine |
| playback_inject | in | Bin stream from playback block | Muxed with live OBG input [SPEC] §8.1 |
| playback_active | in | 1-bit | When high, playback replaces live bins |

**Design decisions needed:**

| # | Decision | Notes |
|---|----------|-------|
| BSd-1 | Mapping table structure and size. How many AxC entries? Max 96 beam slots per SFU [SPEC] §12.1 suggests up to 96 mapping entries. Each entry: {lane_id[1:0], bin_offset[12:0], bin_count[8:0], sub_band[0], slot_index[6:0]}. | [PROP] 96-entry config RAM. ~32 bits/entry. |
| BSd-2 | Is the mapping a sequential scan or a true crossbar? If bins within one OBG lane can be routed to arbitrary non-contiguous sub-band positions, the mux is complex. If bins are routed as contiguous blocks to contiguous sub-band slots, it is a block copy. | [INF] "spectral position (bin slot index) within that sub-band" [SPEC] §6.3 implies contiguous block placement. Propose contiguous-block-copy model. |
| BSd-3 | Can a single OBG lane feed both sub-bands? Not prohibited by spec. If allowed, one OBG lane's bins split across sub-band A and sub-band B. | [INF] Likely needed for deployments where 4 OBG lanes × 5,760 bins > one sub-band (12,288 bins). Must support. |
| BSd-4 | Bin ordering within OBG frame — frequency-ascending? Depends on OBG ICD (not defined). | Blocks detailed RTL. |

---

### 2.5 fb_synth (DL Filter Bank Synthesis) — ×2 sub-band instances

**Source:** SFU §6.4

| Attribute | Value | Origin |
|-----------|-------|--------|
| Architecture | MDFT (Modified DFT) — polyphase FIR + FFT modulation | [SPEC] §6.4 |
| Instances | 2 (sub-band A and sub-band B, independent) | [SPEC] §6.4 |
| Input | 12,288 frequency-domain bins per sub-band per frame | [SPEC] §6.4 |
| Output | 240 time-domain samples per beam slot per frame | [SPEC] §6.4 |
| Parallel lanes | 3 (P_LANE_NR = 3) | [SPEC] §6.4 |
| Beam slots per sub-band | 48 (16 per lane × 3 lanes) | [SPEC] §6.4 |
| FFT core | FLEXFFT240 — mixed-radix 5×4×3×2×2, 5 stages | [SPEC] §6.4 |
| Supported BW modes | 20/10/5/3.33/1.66 MHz (ASIC); 10.24/5.12/2.56 MHz (FPGA) | [SPEC] §6.4 |
| Bins per beam slot | 240/120/60/40/20 (ASIC); 120/60/30 (FPGA) — 2.56 MHz TBD | [SPEC] §6.4 |
| Native BW | 1,024 MHz (ASIC) / 1,048.576 MHz (FPGA) | [SPEC] §6.4 |
| Usable BW | ~819.2 MHz (80%) | [SPEC] §6.4 |
| Input clock | 341.333 MHz | [SPEC] §6.4 |
| Output clock | 320 MHz | [SPEC] §6.4 |
| Internal CDC | Yes — inside the FB IP | [SPEC] §6.4 |
| Clipping detection | Per lane, per FFT stage → reported to management | [SPEC] §6.4 |

**Interfaces:**

| Port | Direction | Width / type | Notes |
|------|-----------|-------------|-------|
| bin_in | in | 12,288-bin frame @ 341.333 MHz | From bins_sel_dl |
| sample_out | out | Time-domain IQ stream @ 320 MHz | To beacon_inj (DL path) |
| beam_slot_bw_cfg[0:47] | in | BW mode per beam slot | From param_engine |
| clip_event | out | Per-lane, per-stage clip flags | To mgmt_regs |

**Open questions:**

| # | Question | Impact |
|---|----------|--------|
| FBS-1 | IP delivery status — custom RTL or sourced from ref [7] (Filter Bank Design Spec by Fryderyk Fijalkowski)? | Critical path for SFU schedule |
| FBS-2 | Polyphase FIR coefficient sets: how many distinct sets (one per BW mode per sat type)? Storage requirements? | Affects coefficient ROM sizing |
| FBS-3 | Is per-beam-slot BW mode truly runtime-selectable, or set at commissioning? §6.4 says "Selected per beam slot at runtime." | If runtime, the FB must support dynamic BW mode switching per slot without pipeline flush. |
| FBS-4 | 240-point FFT with 12,288-bin input: this is a channelizer where 12,288/240 = 51.2 — not an integer. The MDFT polyphase structure resolves this, but the exact overlap/decimation factor is not stated. Defined in ref [7]. | Must obtain ref [7] for implementation. |

---

### 2.6 fb_analysis (UL Filter Bank Analysis) — ×2 sub-band instances

**Source:** SFU §7.4

Architecturally the inverse of fb_synth. Same MDFT architecture, same dimensioning.

| Attribute | Value | Origin |
|-----------|-------|--------|
| Input | 240 time-domain samples/beam slot @ 320 MHz | [SPEC] §7.4 |
| Output | 12,288 frequency-domain bins/sub-band @ 341.333 MHz | [SPEC] §7.4 |
| Clock domains | 320 MHz in / 341.333 MHz out (inverse of DL) | [SPEC] §7.4 |
| Clipping detection | Per lane → reported to management | [SPEC] §7.4 |

Interface and open questions mirror fb_synth. Same IP with inverse configuration is assumed [INF].

---

### 2.7 beacon_inj (CW Beacon Injection — DL only)

**Source:** SFU §6.8

| Attribute | Value | Origin |
|-----------|-------|--------|
| Function | Generate CW tone, add to DL composite signal | [SPEC] §6.8 |
| Injection point | After fb_synth, before band_gain DL | [SPEC] §6.8 — see review note §5.1 item 2 for ambiguity analysis. Most consistent reading. |
| Frequency range | Full available spectrum across both sub-bands | [SPEC] §6.8 |
| Gain range | −60 to +60 dBFs, 0.1 dBFs resolution | [SPEC] §6.8 |
| Gain independence | Independent of Band Gain | [SPEC] §6.8 |
| Enable | Per SFU, UTC-Scheduled | [SPEC] §14.1 |
| Frequency config | Per SFU, UTC-Scheduled, NCO resolution 1 Hz | [SPEC] §14.1 |
| Power config | Per SFU, UTC-Scheduled | [SPEC] §14.1 |
| Always-on | Continuously transmitted when enabled | [SPEC] §6.8 |
| Primary purpose | CPBF satellite residual frequency correction reference | [SPEC] §6.8 |
| Secondary purposes | Band Gain reference, link verification, path loss measurement, Doppler cross-check | [SPEC] §6.8 |

**Design proposal:**

[PROP] Implementation: single NCO generating complex exponential at configured frequency. Amplitude scaled by beacon power register. Output added (complex add) to fb_synth time-domain output before band_gain.

[PROP] If beacon must be placeable across both sub-bands from a single NCO, the NCO must operate in the combined (post-synthesis, pre-DUC) sample domain. If each sub-band has its own beacon instance, the NCO operates per-sub-band. The spec says "Per SFU" for enable/freq/power — implying one beacon across both sub-bands. This needs architectural clarification.

| # | Question | Impact |
|---|----------|--------|
| BCN-1 | One beacon NCO spanning both sub-bands, or one per sub-band? "Full spectrum of both sub-bands" [SPEC] §14.1 and "Per SFU" suggest one beacon that can be placed in either sub-band's spectrum, but the two sub-bands are processed as independent pipelines at this stage. | If one beacon must land in sub-band B while processing is split, the NCO must be instantiated in both sub-bands with only one active at a time based on configured frequency. |
| BCN-2 | NCO tuning range and resolution — stated as 1 Hz in §14.1. Phase accumulator width for 1 Hz resolution at 320 MHz: ⌈log2(320e6)⌉ = 29 bits. Straightforward. | No issue — confirming feasibility. |

---

### 2.8 band_gain (DL and UL instances, ×2 sub-bands each = 4 total)

**Source:** SFU §6.5 (DL), §7.3 (UL), §14.1

| Attribute | Value | Origin |
|-----------|-------|--------|
| Function | Scalar gain applied to entire sub-band signal | [SPEC] §6.5 |
| Scope | Per sub-band (A, B independent), per direction (DL, UL independent) | [SPEC] §6.5, §7.3 |
| Range | −60 to +60 dBFs | [SPEC] §6.5 |
| Resolution | 0.1 dBFs | [SPEC] §6.5 |
| Apply mechanism | UTC-Scheduled (pending/active register pair) | [SPEC] §6.5 |
| AGB actuator | Yes — NMS Band Gain loop writes values here | [SPEC] §6.5 |
| Static config mode | Held at fixed value, AGB updates suppressed | [SPEC] §6.5, §15.1 |
| Post-stage monitoring | Clip detection (saturation alarm) + round detection (bit-width reduction) | [SPEC] §6.5 |
| Telemetry | Saturation event counter (cumulative), alarm on detection | [SPEC] §14.2 |

**DL application point:** After fb_synth (and beacon_inj), before band_dop DL [SPEC] §6.5.  
**UL application point:** After band_dop UL, before fb_analysis [SPEC] §7.3.

**Interfaces:**

| Port | Direction | Width / type | Notes |
|------|-----------|-------------|-------|
| signal_in | in | IQ stream | Time-domain sub-band signal |
| signal_out | out | IQ stream | Gained signal |
| gain_pending | in | Gain config word | From param_engine |
| gain_active | out | Current applied gain | To mgmt_regs (telemetry) |
| apply_trigger | in | pulse | From timing_core (UTC tick match) |
| clip_alarm | out | pulse | To mgmt_regs |
| clip_count | out | counter value | To mgmt_regs (cumulative) |
| round_count | out | counter value | To mgmt_regs (cumulative) |
| static_mode | in | 1-bit | When high, AGB updates blocked |

**Design proposal:**

[PROP] Gain register encoding: 120 dB range at 0.1 dB → 1,201 discrete values → 11-bit unsigned index. Internal implementation: convert dB to linear scale via lookup table (LUT) or CORDIC. LUT is preferred for deterministic latency — 2,048-entry LUT covers the range with interpolation.

[PROP] Multiplier: single complex multiplier (2 real multiplies + 2 adds for IQ). At 320 MHz (post-FB), one multiplier per sub-band per direction is sufficient — the signal is a single wideband stream, not per-beam.

[PROP] Clip detection: compare absolute value of output against maximum representable value (2^(N-1) − 1 for N-bit output). Flag if exceeded. Counter increments on each flagged sample, reported via mgmt_regs.

---

### 2.9 band_dop (DL and UL instances, ×2 sub-bands each = 4 total)

**Source:** SFU §6.6 (DL), §7.2 (UL), §14.1

| Attribute | Value | Origin |
|-----------|-------|--------|
| Function | Frequency shift via NCO complex mixer | [SPEC] §6.6 |
| Scope | Per sub-band (A, B independent), per direction (DL, UL independent) | [SPEC] §6.6 |
| Range | ±512 Hz (ASIC) / ±524.288 Hz (FPGA) | [SPEC] §12.2 |
| Resolution | 1 Hz | [SPEC] §6.6 |
| Operating modes | Operational (1PPS-driven from TLE) / Static (fixed, UTC-Scheduled) | [SPEC] §6.6 |
| Operational mode apply | Local Autonomous — computed from TLE, applied at 1PPS tick | [SPEC] §6.6 |
| Static mode apply | UTC-Scheduled — fixed value via management | [SPEC] §6.6 |
| Mode selection | UTC-Scheduled parameter | [SPEC] §6.6 |
| TLE source | Manager SW → SFU management → PS → PL register | [SPEC] §6.6 |
| DL/UL independence | Independent NCOs, same 1PPS reference, may differ in sign/magnitude | [SPEC] §7.2 |

**DL application point:** After band_gain DL, before DUC/DAC [SPEC] §6.6.  
**UL application point:** After DDC, before band_gain UL [SPEC] §7.2.

**Interfaces:**

| Port | Direction | Width / type | Notes |
|------|-----------|-------------|-------|
| signal_in | in | IQ stream | Time-domain sub-band signal |
| signal_out | out | IQ stream | Frequency-shifted signal |
| doppler_val | in | Signed frequency word | From tle_doppler (operational) or param_engine (static) |
| mode_sel | in | 1-bit | 0 = operational, 1 = static |
| pps_tick | in | pulse | From timing_core — triggers value latch in operational mode |
| static_pending | in | Frequency word | From param_engine (UTC-Scheduled static value) |
| static_apply | in | pulse | From param_engine (UTC tick for static mode) |
| current_freq | out | Signed frequency word | To mgmt_regs telemetry |

**Design proposal:**

[PROP] NCO implementation: phase accumulator + CORDIC or sin/cos LUT. Phase accumulator width: 1 Hz resolution at 320 MHz → 29-bit accumulator (same analysis as beacon). ±512 Hz range is trivially within the NCO capability.

[PROP] Phase-continuous update: at each 1PPS tick, the new frequency word is loaded into the phase increment register. The phase accumulator continues from its current value — no phase discontinuity. This ensures glitch-free Doppler correction transitions.

[PROP] PS/PL handoff: the tle_doppler module on PS (ARM) writes four frequency words (DL_A, DL_B, UL_A, UL_B) into PL-side holding registers via AXI. At the next 1PPS tick, the PL latches all four simultaneously. This ensures all sub-bands update atomically at the same 1PPS edge.

---

### 2.10 duc_ddc

**Source:** SFU §6.7 (DUC), §7.1 (DDC)

| Attribute | Value | Origin |
|-----------|-------|--------|
| DUC interpolation | 4× | [SPEC] §6.7 |
| DDC decimation | 4× | [SPEC] §7.1 |
| DAC sampling rate | 4,096 Msps (ASIC) / 4,194.304 Msps (FPGA) | [SPEC] §9.1 |
| ADC sampling rate | 4,096 Msps (ASIC) / 4,194.304 Msps (FPGA) | [SPEC] §9.1 |
| Resolution | 14-bit ADC and DAC | [SPEC] §9.1 |
| Anti-alias filter | On-chip, ~80% usable BW | [SPEC] §7.1 |

[PROP] DUC: cascade of half-band interpolation filters (2× + 2× = 4×). Standard Xilinx RFSoC DUC IP or custom polyphase. Operating at 320 MHz input → 1,280 MHz intermediate → output at DAC tile rate.

[PROP] DDC: inverse cascade. ADC tile rate → 1,280 MHz intermediate → 320 MHz output.

[INF] The DUC/DDC operates on the composite signal from both sub-bands combined. The two sub-bands must be combined (sub-band A at lower frequency offset, sub-band B at upper offset) before DUC. The sub-band combining point is not explicitly described as a separate block in the SFU doc — it is implicit in the "RF output bandwidth = 1.6 GHz" [SPEC] §6.7. This combining step must be defined.

| # | Question | Impact |
|---|----------|--------|
| DUC-1 | Sub-band A + B combining: at what point are the two sub-bands frequency-shifted to their respective RF offsets and summed? Before DUC or within DUC? | Must define a sub-band combiner block or confirm DUC IP handles dual-sub-band input natively. |
| DUC-2 | RFSoC DAC tile configuration: direct RF sampling at 4,096 Msps, which Nyquist zone? RF centre frequency up to 6 GHz [CTX] §21.5 requires higher Nyquist zone operation. | RF system-level decision, but affects DAC NCO configuration. |

---

### 2.11 rf_port_sel

**Source:** SFU §6.7, §14.1

| Attribute | Value | Origin |
|-----------|-------|--------|
| TX ports | T0 (operational), T1 (redundant 1), T2 (redundant 2) | [SPEC] §10.3 |
| RX ports | R0 (operational), R1 (redundant 1), R3 (redundant 2) | [SPEC] §10.3 |
| Switching | TX and RX switch as a pair | [SPEC] §6.7 |
| Apply mechanism | UTC-Scheduled | [SPEC] §14.1 |
| RF output power | −32 to 0 dBm (all TX ports) | [SPEC] §12.2 |
| RF input power | −80 to 0 dBm (all RX ports) | [SPEC] §12.2 |

[INF] This is a control-logic-only block in the FPGA — the actual RF path switching uses the RFSoC ADC/DAC tile configuration to select which physical RF port pair is active. The FPGA drives the tile mux control signals.

---

### 2.12 timing_core

**Source:** SFU §11.1–§11.5, §6.6, §14.1

| Attribute | Value | Origin |
|-----------|-------|--------|
| 1PPS input | ET port (SSMC), GPS-disciplined, 5 V TTL (TBD OI-S03) | [SPEC] §11.3 |
| 1PPS function | Band Doppler apply trigger, UTC-Scheduled parameter apply tick | [SPEC] §11.3, §17.1 |
| UTC derivation | 1PPS provides 1-second tick; UTC epoch set at commissioning | [CTX] §15.2 |
| Primary clock source | CDR from selected OBG DL Aurora lane | [SPEC] §11.1 |
| Secondary clock source | 10 MHz CI port → PLL → internal clocks | [SPEC] §11.2 |
| Clock select | Pre-configured via NMS, stored NV | [SPEC] §11.4 |
| Auto-switchover | Primary → Secondary on CDR unlock (if configured). Alarm raised. No auto-recovery to primary. | [SPEC] §11.4 |
| 10 MHz CI spec | 1 V, +13 dBm, 50 Ω (secondary mode) | [SPEC] §11.2 |

**Interfaces:**

| Port | Direction | Width / type | Notes |
|------|-----------|-------------|-------|
| pps_in | in | 1-bit (ET port) | Edge-detected to generate internal tick |
| pps_tick | out | 1-cycle pulse in DSP clock domain | Distributed to band_dop, param_engine |
| utc_counter | out | 32-bit or wider UTC second count | For UTC-Scheduled apply comparison |
| clk_sel | in | 1-bit config | From mgmt_regs (Primary/Secondary) |
| clk_active | out | 1-bit status | To mgmt_regs (which clock is active) |
| cdr_unlock_event | in | pulse from aurora_rx_tx | Triggers auto-switchover if configured |
| clk_switchover_alarm | out | alarm | To mgmt_regs |

**Design proposal:**

[PROP] UTC counter: simple 32-bit counter incrementing on each 1PPS tick. UTC epoch offset loaded at commissioning via management. Sufficient for >136 years of unique timestamps.

[PROP] 1PPS edge detector: synchronize ET port input to DSP clock domain (double-FF), detect rising edge, generate single-cycle pps_tick. Jitter from synchronization is ≤2 DSP clock cycles (~6 ns at 341 MHz) — acceptable for 1 Hz update rate.

[PROP] Clock mode logic sits outside the DSP pipeline — it controls the RFSoC PLL/MMCM configuration. Switchover requires PLL relock, which causes a brief clock disruption. This is acceptable as it only occurs on alarm conditions and requires manual recovery.

---

### 2.13 param_engine (UTC-Scheduled Parameter Apply)

**Source:** SFU §17.1, §14.1; CTX §15.2

| Attribute | Value | Origin |
|-----------|-------|--------|
| Function | Generic apply engine: stores pending values with UTC timestamp, applies at matching 1PPS tick, confirms to management | [SPEC] §17.1 |
| Managed parameters | Band Gain DL/UL (×2 sub-bands), RF port select, CW Beacon (enable/freq/power), Band Gain loop enable, clock mode, static config mode, phase alignment delays, filter bank config, commissioning params | [SPEC] §14.1 |
| Apply flow | (1) Manager SW writes {value, utc_timestamp} → pending register. (2) At matching 1PPS-derived UTC tick, pending → active. (3) Confirmation sent to Manager SW. | [SPEC] §17.1 |
| Transition effect | Accepted — in-flight OBG samples may cross the apply boundary | [SPEC] §17.1 |

**Design proposal:**

[PROP] Structure: array of parameter slots, each containing:
```
struct param_slot {
    logic [DATA_W-1:0]  pending_value;
    logic [31:0]        apply_utc;        // UTC second to apply
    logic               pending_valid;    // Slot has a pending update
    logic [DATA_W-1:0]  active_value;     // Currently applied value
    logic               apply_confirmed;  // Confirmation flag for readback
};
```

At each 1PPS tick, the engine scans all slots: if `pending_valid && (utc_counter == apply_utc)`, swap `pending_value → active_value`, clear `pending_valid`, set `apply_confirmed`. The management interface reads `apply_confirmed` to generate the confirmation message.

[PROP] Number of slots: enumerate all UTC-Scheduled parameters from §14.1 → approximately 15–20 distinct parameters. A 32-slot array provides headroom.

[PROP] Multiple pending updates: the spec does not address queuing multiple future updates for the same parameter. Simplest: one pending slot per parameter. If a new write arrives while a previous pending is not yet applied, overwrite the pending. This matches "pre-loaded with a future UTC timestamp" wording.

---

### 2.14 mgmt_regs

**Source:** SFU §14.1, §14.2, §17.1, §17.2

This is the central register file accessible from the PS (ARM) via AXI. It aggregates all controllable parameters and telemetry points.

**Register groups:**

| Group | Registers | R/W | Source block |
|-------|-----------|-----|-------------|
| Band Gain DL | gain_pending, gain_active, apply_utc (×2 sub-bands) | R/W pending; RO active | band_gain DL |
| Band Gain UL | Same structure (×2 sub-bands) | R/W / RO | band_gain UL |
| Band Doppler DL | doppler_val, mode_sel, static_pending (×2 sub-bands) | R/W | band_dop DL |
| Band Doppler UL | Same (×2 sub-bands) | R/W | band_dop UL |
| RF port select | port_sel_pending, port_sel_active, apply_utc | R/W / RO | rf_port_sel |
| CW Beacon | enable, freq, power, apply_utc | R/W | beacon_inj |
| Band Gain loop | enable per sub-band | R/W | band_gain |
| Clock | clk_mode, clk_active, cdr_lock, switchover_alarm | R/W / RO | timing_core |
| Static config | static_mode_enable | R/W | global |
| Lane alignment | delay_reg[0:3], master_lane_sel | R/W | lane_align |
| Aurora status | lane_status[0:3], cdr_lock | RO | aurora_rx_tx |
| FB clipping | clip_count_dl[0:2], clip_count_ul[0:2] per sub-band | RO | fb_synth, fb_analysis |
| Band Gain saturation | clip_count_dl, clip_count_ul per sub-band | RO | band_gain |
| Band RMS power | rms_dl, rms_ul per sub-band | RO | rms_detector |
| Pipeline latency | dl_latency_cycles, dl_latency_ns, ul_latency_cycles, ul_latency_ns | RO | latency_meas |
| FPGA temperature | junction_temp_c | RO | RFSoC SYSMONE4 |
| Firmware version | fw_version_string | RO | Build-time constant |
| Band RMS window | rms_window_log2 (range 2 to 10) | R/W | rms_detector |
| Phase alignment measurement | per-lane measured delay | RO | lane_align |
| Capture control | capture_trigger, capture_point, capture_status | R/W / RO | capture |
| Playback control | pb_trigger, pb_bw_mode, pb_loop, pb_status | R/W / RO | playback |
| Event log | log read port | RO | event_log |

**Design proposal:**

[PROP] AXI4-Lite slave on the PS-PL interface. 32-bit register width. Address space partitioned by block with reserved ranges for future expansion. Read/write arbitration: PS writes to pending registers; PL writes to status/telemetry registers. No contention — clearly partitioned R/W ownership.

---

### 2.15 rms_detector (Band RMS Power Measurement)

**Source:** SFU §14.2, §12.2

| Attribute | Value | Origin |
|-----------|-------|--------|
| Measurement | Band RMS power | [SPEC] §14.2 |
| Scope | Per sub-band, per direction (DL after Band Gain, UL before Band Gain) | [SPEC] §14.2 |
| Resolution | ~1 dB | [SPEC] §14.2 |
| Window range | 2^2 to 2^10 samples (max/default 1,024) | [SPEC] §12.2 |
| Used by | Band Gain loop (AGB) | [SPEC] §14.2 |

[PROP] Implementation: accumulate |I|^2 + |Q|^2 over configurable window, compute mean, convert to dBFs. Use approximation (e.g., log2 via leading-zero count) for ~1 dB resolution. Instances: 4 (DL_A, DL_B, UL_A, UL_B).

[INF] The DL measurement point is "after Band Gain" [SPEC] §14.2. The UL measurement point is "before Band Gain UL" [SPEC] §14.2. These are at different points in the respective chains. For the UL, this means the RMS detector sees the signal as received (post-Doppler, pre-Gain), which is the correct feedback point for an input-level-based AGB loop.

---

### 2.16 latency_meas

**Source:** SFU §13, §14.2, SFU-LAT-04

| Attribute | Value | Origin |
|-----------|-------|--------|
| Function | Measure DL and UL pipeline latency in DSP cycles and ns | [SPEC] §13, SFU-LAT-04 |
| Update | On any CDC FIFO correction event | [SPEC] §14.2 |
| Requirement | Fixed, deterministic, identical across same-HW-rev/FW-ver cards | [SPEC] §13, SFU-LAT-01 to -03 |

[PROP] Implementation: embed a known marker (e.g., frame counter or timestamp) at the OBG input and detect it at the DAC output (DL) or embed at ADC input and detect at OBG output (UL). The difference in timestamps = pipeline latency. Alternative: count clock cycles from marker insertion to marker detection using a shared free-running counter.

[PROP] On CDC correction events, the latency changes by exactly 1 frame period (deterministic). Update the reported value accordingly.

---

### 2.17 playback

**Source:** SFU §8.1

| Attribute | Value | Origin |
|-----------|-------|--------|
| Injection point | Pre-Bins Selector DL — replaces live Aurora OBG input | [SPEC] §8.1 |
| Data domain | Frequency-domain bins (OBG bin format) | [INF] from injection point being pre-Bins Selector |
| Supported BWs | 1.66 / 3.33 / 5 / 10 / 20 MHz | [SPEC] §8.1 |
| Min duration | 10 ms (1 LTE frame) | [SPEC] §8.1 |
| LTE frame alignment | Yes — triggered at LTE frame boundary | [SPEC] §8.1 |
| Loop mode | Single-shot (10 ms) or continuous | [SPEC] §8.1 |
| Trigger | Immediate / time-aligned / 1PPS-aligned | [SPEC] §8.1 |
| Memory depth | TBD (OI-S05) | [TBD] |
| Traffic impact | Interrupts live DL — debug only | [SPEC] §8.1 |

**Design proposal:**

[PROP] Memory: URAM-based buffer holding bin-domain IQ data. For a 5 MHz cell (60 bins, 10 ms), assuming 32-bit IQ (16I+16Q) per bin at 83.333 kHz sample rate: 60 bins × 12,000 samples/s × 0.01 s = 7,200 samples × 4 bytes = 28.8 KB. For 20 MHz (240 bins): 115.2 KB. Manageable in URAM.

[PROP] Loading: test vectors loaded via management interface (1 GbE). Stored in URAM. Playback controller replays from URAM into bins_sel_dl input mux when triggered.

---

### 2.18 capture

**Source:** SFU §8.2

| Attribute | Value | Origin |
|-----------|-------|--------|
| Capture points | Post-Bins Selector DL, post-FB Analysis UL, others TBD (OI-S08) | [SPEC] §8.2 |
| Min duration | 10 ms | [SPEC] §8.2 |
| Target duration | Full satellite pass (best effort) | [SPEC] §8.2 |
| Trigger modes | On-demand, time-triggered (UTC), event-triggered (CDC/CDR/AGB), continuous (ring buffer) | [SPEC] §8.2 |
| Download | Via 1 GbE management, non-interrupting | [SPEC] §8.2 |
| Sample format | IQ (width TBD) | [SPEC] §8.2 |
| Memory depth | TBD (OI-S08) | [TBD] |

[PROP] Architecture: multiplexed capture bus with selectable tap points. BRAM/URAM ring buffer. DMA to PS DRAM for large captures. Download from PS DRAM via 1 GbE. Trigger state machine with UTC comparator, event input port, and manual trigger.

---

### 2.19 spur_det (Spur Detection)

**Source:** SFU §15.6

| Attribute | Value | Origin |
|-----------|-------|--------|
| Path | UL — post FB Analysis output | [SPEC] §15.6 |
| Measurement | Per-bin RMS power, 83.333 kHz resolution | [SPEC] §15.6 |
| Operation | Cyclic autonomous scan, non-interrupting | [SPEC] §15.6 |
| Output | Per-bin power array (bin index → dBFs) | [SPEC] §15.6 |
| Update rate | TBD (OI-S10) | [TBD] |
| Reporting | Via management on request or at configured interval | [SPEC] §15.6 |
| Resource sharing with spectrum_monitor | TBD (OI-S10) | [TBD] |

[PROP] Implementation: single |I|^2 + |Q|^2 accumulator that cycles through bins sequentially. At 83.333 kHz bin rate and 19,200 occupied bins, one full scan takes 19,200 / 83,333 ≈ 230 ms if one bin is measured per bin-period. Result stored in a BRAM-based power array readable by management.

---

## 3. Clock Domain Summary

| Clock domain | Frequency | Source | Used by |
|--------------|-----------|--------|---------|
| aurora_clk | Recovered from CDR (≈ line rate / encoding ratio) | Aurora GT transceiver | aurora_rx_tx, dl_cdc source side, ul_cdc dest side |
| dsp_clk_341 | 341.333 MHz | PLL from aurora_clk (primary) or 10 MHz (secondary) | dl_cdc dest, lane_align, bins_sel_dl, fb_synth input, fb_analysis output, bins_sel_ul, ul_cdc source |
| dsp_clk_320 | 320 MHz | PLL from aurora_clk (primary) or 10 MHz (secondary) | fb_synth output, beacon_inj, band_gain, band_dop, rms_detector, duc_ddc input, fb_analysis input |
| dac_tile_clk | 4,096 / 4,194.304 MHz (or sub-harmonic) | RFSoC DAC tile PLL | DUC final stage, DAC interface |
| adc_tile_clk | 4,096 / 4,194.304 MHz (or sub-harmonic) | RFSoC ADC tile PLL | ADC interface, DDC first stage |
| axi_clk | ~100–250 MHz (RFSoC PS-PL AXI) | PS clock | mgmt_regs, param_engine, capture DMA, playback load |
| pps_async | 1 Hz, asynchronous | ET port (external GPS) | timing_core input (synchronised internally) |

**CDC crossings identified:**

| Crossing | From → To | Block | Mechanism |
|----------|-----------|-------|-----------|
| CDC-A | aurora_clk → dsp_clk_341 | dl_cdc | Async FIFO per lane |
| CDC-B | dsp_clk_341 → aurora_clk | ul_cdc | Async FIFO per lane |
| CDC-C | dsp_clk_341 → dsp_clk_320 | Inside fb_synth IP | Internal to FB IP |
| CDC-D | dsp_clk_320 → dsp_clk_341 | Inside fb_analysis IP | Internal to FB IP |
| CDC-E | dsp_clk_320 → dac_tile_clk | DUC output stage | RFSoC tile interface FIFO |
| CDC-F | adc_tile_clk → dsp_clk_320 | DDC input stage | RFSoC tile interface FIFO |
| CDC-G | pps_async → dsp_clk_341 | timing_core | Double-FF synchroniser + edge detect |
| CDC-H | axi_clk → dsp_clk_* | param_engine, mgmt_regs | Handshake or dual-clock register |

---

## 4. Control and Management Architecture

### 4.1 Parameter apply taxonomy

| Parameter | Instances | Apply mechanism | PL block | PS role |
|-----------|-----------|----------------|----------|---------|
| Band Gain DL | 2 (sub-band A, B) | UTC-Scheduled | band_gain | Relay from Manager SW |
| Band Gain UL | 2 | UTC-Scheduled | band_gain | Relay |
| Band Doppler DL | 2 | Local Autonomous (operational) or UTC-Scheduled (static) | band_dop | TLE → Doppler compute (operational); relay (static) |
| Band Doppler UL | 2 | Local Autonomous / UTC-Scheduled | band_dop | Same |
| RF port select | 1 | UTC-Scheduled | rf_port_sel | Relay |
| CW Beacon enable | 1 | UTC-Scheduled | beacon_inj | Relay |
| CW Beacon freq | 1 | UTC-Scheduled | beacon_inj | Relay |
| CW Beacon power | 1 | UTC-Scheduled | beacon_inj | Relay |
| Band Gain loop enable | 2 | UTC-Scheduled | band_gain | Relay |
| Clock mode | 1 | UTC-Scheduled | timing_core | Relay + PLL reconfig |
| Static config mode | 1 | UTC-Scheduled | global | Relay |
| Lane alignment delay | 4 (per lane) | UTC-Scheduled (commissioning) | lane_align | Relay |
| FB beam slot BW config | 96 (per beam slot) | UTC-Scheduled (commissioning) | fb_synth, fb_analysis | Relay |
| Bins Selector mapping | ~96 entries | UTC-Scheduled (commissioning) | bins_sel_dl, bins_sel_ul | Relay |
| RMS window | 1 | UTC-Scheduled | rms_detector | Relay |

### 4.2 PS software modules

| Module | Function | Interface to PL |
|--------|----------|----------------|
| mgmt_sw | 1 GbE agent: receives Manager SW commands, translates to register writes, sends telemetry/confirmations upstream | AXI4-Lite to mgmt_regs |
| tle_doppler | SGP4/SDP4 orbital propagator. Receives TLE from Manager SW, computes Doppler per sub-band per direction at ~1 Hz, writes to PL holding registers before each 1PPS | AXI4-Lite to band_dop holding regs |
| nv_config | Flash read/write for non-volatile parameter storage and restore on startup | SPI flash or QSPI |
| fw_mgmt | Firmware upgrade agent | PCAP / ICAP interface for partial/full reconfig |
| event_log_mgr | Reads PL event FIFO, formats, stores, serves to management interface | AXI4-Lite or AXI4-Stream from PL event FIFO |

### 4.3 PS/PL handoff for Band Doppler (operational mode)

This is the most timing-critical PS→PL interaction.

```
PS (ARM):                          PL (FPGA):
                                   
  TLE data from Manager SW         
  ↓                                
  SGP4 compute (< 100 ms)         
  ↓                                
  Write doppler_dl_a,              
        doppler_dl_b,              
        doppler_ul_a,              
        doppler_ul_b               
  to PL holding registers          
  via AXI4-Lite                    
  ↓                                
  Set "doppler_ready" flag ──────▶ PL sees doppler_ready
                                   ↓
                                   Wait for 1PPS tick
                                   ↓
                                   Latch all 4 values
                                   simultaneously into
                                   active NCO freq regs
                                   ↓
                                   Clear doppler_ready
                                   ↓
                                   Report via telemetry
```

**Timing constraint:** PS must complete SGP4 + AXI writes within the 1-second 1PPS interval. SGP4 on ARM A53 at ~1 GHz takes <10 ms for a single propagation. Margin is large.

**Failure mode:** If PS does not set `doppler_ready` before the 1PPS tick, PL holds the previous Doppler value. An alarm is raised ("Doppler update stale"). [PROP] — not explicitly specified but required for robustness.

---

## 5. Debug and Telemetry Architecture

### 5.1 Telemetry points (from SFU §14.2)

| Telemetry | Block source | Rate | Format |
|-----------|-------------|------|--------|
| Band RMS power DL (×2) | rms_detector | ≤1 s | dBFs |
| Band RMS power UL (×2) | rms_detector | ≤1 s | dBFs |
| Current Band Gain DL/UL (×4) | band_gain | On change | dBFs |
| DL/UL pipeline latency | latency_meas | On CDC event | cycles + ns |
| FPGA junction temperature | RFSoC SYSMONE4 | ≤1 s | °C |
| Active clock source | timing_core | On change | enum |
| CDR lock status | aurora_rx_tx | On change (alarm) | bool |
| Aurora lane status (×4) | aurora_rx_tx | On change (alarm) | bool |
| Band Gain error signal (×2) | rms_detector / band_gain | ≤1 s | dB |
| FB clipping DL/UL (×6 lanes) | fb_synth / fb_analysis | ≤1 s (cumulative) | count |
| Band Gain DL saturation (×2) | band_gain | On event (alarm) | count |
| Band Gain UL saturation (×2) | band_gain | On event (alarm) | count |
| Firmware version | Constant | On request | string |

### 5.2 Alarm events

| Alarm | Source | Trigger |
|-------|--------|---------|
| CDR unlock | aurora_rx_tx | CDR loses lock |
| Aurora lane loss | aurora_rx_tx | Any lane transitions Up → Down |
| Band Gain DL saturation | band_gain DL | Clip detection fires |
| Band Gain UL saturation | band_gain UL | Clip detection fires |
| FB clipping | fb_synth / fb_analysis | Per-lane per-stage clip |
| Clock switchover | timing_core | Primary → Secondary switch |
| UTC-Scheduled apply failure | param_engine | Pending not applied within expected tick |
| Doppler update stale | band_dop | [PROP] doppler_ready not set before 1PPS |
| CDC correction | dl_cdc / ul_cdc | Pointer adjustment event |

### 5.3 Debug modes

| Mode | Described in | Blocks affected | Key behavior |
|------|-------------|----------------|--------------|
| Static config | §15.1 | band_gain, band_dop (all instances) | All dynamic updates frozen. Band Doppler NCO held constant. AGB loop disabled. |
| Playback | §8.1 | bins_sel_dl input mux | Pre-recorded bins replace live Aurora. Interrupts traffic. |
| Capture | §8.2 | Configurable tap points | IQ recording to buffer, multiple trigger modes. Non-interrupting (read via mgmt). |
| Spur detection | §15.6 | UL post-FB Analysis | Cyclic per-bin RMS scan. Non-interrupting. |
| Spectrum monitor | §15.5 | TBD | Real-time power-per-bin display. Non-interrupting. TBD (OI-S10). |
| Loopback | §15.3 | TBD | Digital DL→UL or RF loopback. Feasibility TBD (OI-S09). |

---

## 6. Open Implementation Questions

### 6.1 Critical — blocks RTL start

| # | Question | OI ref | Proposed action |
|---|----------|--------|----------------|
| IQ-1 | OBG Aurora frame format (header, bin ordering, metadata, framing, CRC) | None | Author joint ICD with DCU team. Priority 1. |
| IQ-2 | Filter bank IP source, delivery date, integration guide, coefficient format | Ref [7] | Obtain from DSP Lead. Priority 1. |
| IQ-3 | Management register map / protocol (REST? raw register? SNMP agent?) | OI-S07 | Define AXI register map internally now (protocol-agnostic). PS software adapter designed separately. |
| IQ-4 | Single or dual FPGA bitstream for ASIC/FPGA satellite | None | Architecture decision needed. Recommend single bitstream with PLL reconfig. |

### 6.2 High — affects block sizing or interface

| # | Question | OI ref | Notes |
|---|----------|--------|-------|
| IQ-5 | Bins Selector mapping constraints: can one OBG lane feed both sub-bands? Full crossbar vs constrained routing? | None | Propose constrained: each lane → one sub-band only. Seek confirmation. |
| IQ-6 | CW Beacon: one NCO per SFU (spanning both sub-bands) or one per sub-band? | None | Propose one instance per sub-band, only one active based on configured frequency. |
| IQ-7 | Sub-band A + B combining point before DUC: explicit combiner block or handled by RFSoC tile multi-band config? | None | Investigate RFSoC multi-band DAC mode. |
| IQ-8 | IQ word width at each pipeline stage (Aurora → Bins Selector → FB → Band Gain → Band Doppler → DUC → DAC) | None | Define bit-growth budget through chain. Needed for BRAM/multiplier sizing. |
| IQ-9 | Playback data format: bin-domain IQ confirmed? Word width? Memory depth? | OI-S05 | Confirm bin-domain. Allocate URAM budget conservatively. |
| IQ-10 | Capture memory depth and DMA architecture to PS DRAM | OI-S08 | Determine max capture rate vs 1 GbE download bandwidth. |
| IQ-11 | Inter-SFU phase skew budget → lane_align delay register depth | OI-S02 | Propose 64-deep BRAM buffer per lane as conservative default. |
| IQ-12 | CDC FIFO depth for secondary clock mode (larger frequency offset possible) | None | Characterize worst-case offset between independent 10 MHz source and OBG clock. |

### 6.3 Medium — affects verification or debug

| # | Question | OI ref | Notes |
|---|----------|--------|-------|
| IQ-13 | Spur detection vs spectrum monitor resource sharing | OI-S10 | Both read UL FB analysis output. Propose single scan engine with dual reporting modes. |
| IQ-14 | Loopback mode feasibility — digital DL→UL vs RF loopback at RFSoC tile level | OI-S09 | Digital loopback (post-Band Doppler DL → pre-Band Doppler UL) is simplest. RF loopback needs analog path. |
| IQ-15 | Event log: depth, format, persistent storage mechanism | OI-S08 | Propose PL-side FIFO (256 entries), drained by PS into DRAM or flash. |
| IQ-16 | Latency measurement mechanism: marker-based or cycle-counting? | None | Propose frame-counter-based: embed counter at OBG input, read at DAC output, diff = latency. |
| IQ-17 | Band Gain error signal computation: is target - measured computed in PL or PS? | None | Propose PL: simple subtractor on rms_detector output vs configurable target register. |

---

## 7. Resource Estimation Sketch

Preliminary resource awareness for the XCZU43DR-2FFVE1156E (Zynq UltraScale+ RFSoC):

| Resource | Estimated usage | Driver |
|----------|----------------|--------|
| BRAM (36Kb) | ~200–400 | CDC FIFOs, lane_align buffers, playback, capture, coefficient ROM, spur_det power array, event log |
| URAM (288Kb) | ~40–80 | Playback buffer (primary consumer), capture ring buffer, FB coefficient storage if large |
| DSP48E2 | ~200–400 | FB synthesis/analysis (dominant), Band Gain multipliers (4×), Band Doppler NCO CORDIC or LUT multipliers (4×), RMS accumulators, DUC/DDC filters |
| LUT / FF | Moderate | Control logic, Bins Selector routing, param_engine, mgmt_regs, timing_core |
| GTY transceivers | 4 | Aurora 4-lane (Q0 QSFP28) |
| ADC tiles | 3 (of 4 available) | R0, R1, R3 RF Rx |
| DAC tiles | 3 (of 4 available) | T0, T1, T2 RF Tx |

**Note:** The filter bank is the dominant resource consumer. Without the FB IP resource report, this estimate has high uncertainty. The XCZU43DR has 3,024 DSP48E2 slices, 720 BRAM36, and 80 URAM288 — the budget appears feasible but tight depending on FB implementation.

---

*End of micro-architecture document. Next step: resolve IQ-1 through IQ-4 (critical blockers), then proceed to detailed RTL block specifications starting with param_engine, band_gain, band_dop, and timing_core (fully specified, unblocked).*
