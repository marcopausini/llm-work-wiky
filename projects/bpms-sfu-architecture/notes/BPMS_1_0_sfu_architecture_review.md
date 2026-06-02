# BPMS 1.0 — SFU Architecture Review Note

**Document:** Engineering review of BPMS-1.0-ARCH-001 v2.4 and BPMS-1.0-SFU-001 v1.6  
**Date:** April 2026  
**Reviewer role:** FPGA architect / DSP engineer — SFU micro-architecture preparation  
**Purpose:** Extract implementable facts, identify gaps, and establish what can be locked now vs. what is blocked.

---

## 1. system_architecture_summary

BPMS 1.0 is a ground-based LEO satellite gateway that bridges standard LTE eNodeBs (CPRI) to QV-band RF feeds toward LEO bent-pipe satellites. The system compensates Doppler shift, propagation delay, and frequency conversion transparently — the eNodeB and UE are unaware of the satellite link.

The signal chain has five hardware layers:

| Layer | Device | Hardware | Key role |
|-------|--------|----------|----------|
| L1 | eNodeB | LTE base station | 32 AxC/unit, CPRI optical out |
| L2 | DCU | AMD Alveo V80 (PCIe) | CPRI termination, FFT analysis, beam-level corrections (Beam Gain, Beam Doppler, beam delay), OBG fill-and-spill packing. Dual filter bank (ASIC + FPGA sat). 12 CPRI in / 4 OBG out per card. |
| L3 | Optical Matrix | Polatis 6000i | Passive L1 non-blocking crossbar. Transparent to OBG/Aurora. < 1 µs. |
| L4 | **SFU** | BittWare RFX-8440A (RFSoC) | OBG ingest → filter bank synthesis/analysis → Band Gain → Band Doppler → DAC/ADC → RF. Band-level processing only. Satellite-oriented (one sat type at a time). |
| L5 | G-SFU + QV Antenna | 1–6 SFUs per group | Logical grouping; one QV antenna + one satellite per group. Up to 2 pols, max 3 SFU/pol, 9 GHz aggregate cap. |

Gateway scale: up to 5 operational satellites (5 G-SFU groups), 2 shared redundant antennas. Phase 1 interconnect uses Polatis optical matrix; Phase 2 targets 100 GbE Ethernet switch.

The fundamental signal unit is the spectral bin (83.333 kHz ASIC, 85.333 kHz FPGA). All routing, packing, and switching operate at bin granularity. One OBG link carries 5,760 bins and supports mixed-BW AxC packing.

---

## 2. sfu_functional_scope

### 2.1 Ownership boundary — what the SFU does

The SFU is exclusively a **band-level** processing device. It operates on 1,024 MHz sub-bands, applying one correction value per sub-band shared across all beams. Specifically:

- **OBG ingest:** 4 Aurora SERDES lanes via Q0 QSFP28 (19.2 Gbps/lane). CDR clock recovery (primary mode). CDC FIFO at Aurora/DSP clock boundary.
- **Multi-lane phase alignment:** compensate differential optical path + PCB routing skew across 4 OBG lanes before bin routing.
- **Bins Selector (DL):** NMS-configured mapping of {OBG lane, bin offset, count} → {sub-band instance, spectral position}. Bandwidth-agnostic.
- **Filter Bank Synthesis (DL):** MDFT polyphase + FFT. 1,024 MHz native BW/sub-band, ~819.2 MHz usable. 3 parallel lanes, 48 beam slots/sub-band, 96 total. Mixed-radix 240-point FFT (5×4×3×2×2).
- **Band Gain (DL):** one register per sub-band. −60 to +60 dBFs, 0.1 dBFs. UTC-Scheduled. AGB actuator. Clip/round detection post-stage.
- **Band Doppler (DL):** one NCO per sub-band. ±512 Hz (ASIC) / ±524.288 Hz (FPGA). 1 Hz res. Local Autonomous — computed from TLE, applied at 1PPS.
- **CW Beacon:** continuous tone injection in DL after F.B. synthesis, before Band Doppler and Band Gain. Full-spectrum placeable. −60 to +60 dBFs. Functional requirement for satellite CPBF residual correction.
- **RF port select + DUC/DAC:** 4× interpolation, 4,096 Msps (ASIC) / 4,194.304 Msps (FPGA), 14-bit. 3 TX ports (1 operational + 2 hot standby). UTC-Scheduled port switch (TX+RX paired).
- **UL path:** reverse of DL — ADC/DDC → Band Doppler UL → Band Gain UL → F.B. Analysis → Bins Selector UL → OBG Aurora output.
- **Debug:** Playback (inject test bins pre-Bins Selector), Capture (IQ at configurable pipeline points), Static Config Mode, Spur Detection (per-bin RMS on UL), Spectrum Monitor.

### 2.2 Ownership boundary — what the SFU does NOT do

- No per-beam (per-AxC) corrections: Beam Gain, Beam Doppler, beam delay — all DCU.
- No CPRI termination — DCU handles CPRI.
- No OBG packing / fill-and-spill — DCU.
- No filter bank mode selection per OBG — DCU dual filter bank selects ASIC vs FPGA bin width before OBG output.
- No inter-device control channel — OBG is pure data plane.
- No NMS logic — Manager SW is external; SFU receives commands over 1 GbE management.

### 2.3 Satellite type impact on SFU

The SFU is satellite-oriented (one type at a time per G-SFU). The DCU dual filter bank (OI-023 resolved, Option A selected) delivers natively aligned bins for both sat types. From the SFU perspective the primary differences are:

| Parameter | ASIC satellite | FPGA satellite |
|-----------|---------------|----------------|
| ADC/DAC rate | 4,096 Msps | 4,194.304 Msps (×1.024) |
| Bin width | 83.333 kHz | 85.333 kHz |
| F.B. native BW | 1,024 MHz | 1,048.576 MHz |
| Band Doppler range | ±512 Hz | ±524.288 Hz |
| Supported LTE BWs | 20/10/5/3.33/1.66 MHz | 10.24/5.12/2.56 MHz (no 20 MHz) |

**Inference:** The ×1.024 scaling implies different PLL configuration and potentially different filter bank coefficients. The FPGA must support both clock rates, switchable at commissioning. This is not a runtime hot-switch — it requires re-commissioning.

---

## 3. sfu_signal_chain_understanding

### 3.1 Downlink path (OBG → RF)

```
OBG Aurora Rx (4 lanes, Q0)
  → CDR clock recovery (primary mode: one lane = timing master)
  → DL CDC FIFO (Aurora clock → DSP clock domain)
  → Multi-Lane Phase Alignment (per-lane programmable delay)
  → Bins Selector DL (NMS-configured: lane/offset/count → sub-band/position)
  → Filter Bank Synthesis (MDFT, 2 sub-bands × 3 lanes × 48 beam slots)
  → Band Gain DL (per sub-band, UTC-Scheduled, clip/round detect post-stage)
  → CW Beacon injection (after F.B. synthesis, before Band Doppler — per §6.8 SFU doc)
  → Band Doppler DL (per sub-band NCO, Local Autonomous at 1PPS)
  → DUC (4× interpolation)
  → DAC (14-bit, 4,096 / 4,194.304 Msps)
  → RF Port Select (T0/T1/T2, UTC-Scheduled)
  → QV antenna
```

**Note on CW Beacon placement — document inconsistency identified:**
- SFU doc §6.8 states beacon is injected "after the Filter Bank Synthesis block and BEFORE the Band Doppler correction and Band Gain blocks, so the beacon tone passes through both Band Doppler and Band Gain processing."
- SFU doc §6.5 states Band Gain application point is "After F.B Synthesis, before Band Doppler DL."
- This implies the DL order is: F.B. Synthesis → Beacon injection → Band Gain → Band Doppler. But §6.8's own language ("BEFORE the Band Doppler correction and Band Gain blocks") suggests beacon is before both.
- However, the same §6.8 also says the beacon's secondary use is as a "Band Gain reference signal" — if beacon is before Band Gain, then the satellite sees beacon × Band Gain, which is the expected behavior for a reference.
- **Most consistent reading:** F.B. Synthesis → CW Beacon → Band Gain DL → Band Doppler DL → DUC/DAC. The beacon passes through both Band Gain and Band Doppler before reaching RF. This is functionally correct for satellite CPBF usage.

### 3.2 Uplink path (RF → OBG)

```
QV antenna
  → RF Port Select (R0/R1/R3)
  → ADC (14-bit, 4,096 / 4,194.304 Msps)
  → DDC (4× decimation)
  → Band Doppler UL (per sub-band NCO, Local Autonomous at 1PPS)
  → Band Gain UL (per sub-band, UTC-Scheduled, clip/round detect post-stage)
  → Filter Bank Analysis (MDFT, inverse of DL)
  → Bins Selector UL (extract assigned bins, pack into OBG frames)
  → UL CDC FIFO
  → OBG Aurora Tx (4 lanes, Q0 full-duplex)
```

### 3.3 Filter bank architecture

Both DL and UL use MDFT (Modified DFT) filter banks — polyphase FIR pre/post filter + FFT modulation stage.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Sub-bands per SFU | 2 (A lower, B upper) | Processed independently |
| Parallel lanes per sub-band | 3 (P_LANE_NR = 3) | |
| Beam slots per sub-band | 48 (16/lane × 3 lanes) | |
| Total beam slots per SFU | 96 | |
| Bins per sub-band | 12,288 | 3 lanes × 4,096-point FFT |
| Total bins per SFU | 24,576 | |
| Occupied RF bins (usable) | 19,200 | 1,600 MHz ÷ 83.333 kHz |
| FFT size | 240-point (FLEXFFT240) | Mixed-radix: 5×4×3×2×2 |
| DL clocks | 341.333 MHz in / 320 MHz out | CDC inside the IP |
| UL clocks | 320 MHz in / 341.333 MHz out | Inverse of DL |

**Fact:** The 12,288 bins/sub-band figure (3 × 4,096) and the 240-point FFT output frame length are both specified. The relationship between these two numbers defines the filter bank channelization structure. 12,288 input bins at 83.333 kHz spacing = 1,024 MHz, consistent. The 240-sample output frame is per beam slot.

### 3.4 Key interfaces

| Interface | Type | Rate | SFU function |
|-----------|------|------|-------------|
| Q0 (QSFP28) | 4× Aurora SERDES, bidir | 4 × 19.2 Gbps | OBG data (5,760 bins/lane). Breakout to 4× LC duplex for Polatis ports. |
| T0/R0 | SSMC (labeled SMA) | 1.6 GHz BW | Operational RF TX/RX |
| T1/T2, R1/R3 | SSMC | 1.6 GHz BW | Redundant hot-standby RF |
| ET | SSMC | 1 Hz pulse | 1PPS — Band Doppler apply trigger |
| CI | SSMC | 10 MHz sine | Secondary clock mode input |
| CO | SSMC | 10 MHz | Clock output (not used in baseline) |
| RJ-45 (rear) | 1 GbE | — | Management: parameter writes, telemetry, captures, FW |
| USB | Micro-USB | — | FPGA debug/programming (bring-up only) |

---

## 4. key_requirements_and_constraints

### 4.1 Timing and synchronisation

- **Primary clock:** CDR from one pre-selected OBG DL Aurora lane. Drives all FPGA logic + ADC/DAC. No clock shared between SFU cards.
- **Secondary clock:** External 10 MHz at CI port → PLL → all internal clocks. SFU propagates clock upstream via OBG UL.
- **1PPS (ET port):** GPS-disciplined. Provides UTC tick for Band Doppler autonomous apply and UTC-Scheduled parameter activation.
- **Inter-SFU phase coherence:** required within G-SFU group (SYS-CLK-06). In primary mode, inherent from common DCU clock source. Residual skew from differential fibre length — max tolerated skew TBD (OI-S02).
- **Intra-SFU lane alignment:** 4 OBG lanes have differential skew from optical matrix + PCB routing. Per-lane programmable delay register compensates. Set at commissioning.

### 4.2 Control and management

- **UTC-Scheduled parameters:** Band Gain DL/UL, RF port select, CW Beacon (enable/freq/power), AGB loop enable, clock mode, static config mode, filter bank config, commissioning params. Mechanism: Manager SW pre-loads value + future UTC timestamp → device stores in pending register → applies at 1PPS-derived UTC tick → confirms to Manager SW.
- **Local Autonomous parameters:** Band Doppler NCO DL/UL. Manager SW supplies TLE ephemeris. SFU computes correction internally and applies at 1PPS. Manager SW never writes Doppler values directly.
- **No in-band OBG control channel.** All control is out-of-band via 1 GbE management.
- **Transition effect accepted:** at UTC-Scheduled apply instant, in-flight OBG samples may cross parameter boundary. Non-significant by design for these parameter types.

### 4.3 Key architectural rules

1. **sfuPG uniform:** all G-SFU groups at a gateway share the same sfuPG value (1–6).
2. **OBG not shared across SFUs:** each OBG link connects exactly one DCU output to exactly one SFU input.
3. **Mixed-BW OBG:** a single OBG lane carries AxCs of different bandwidths. SFU Bins Selector is BW-agnostic — it maps bin ranges.
4. **Satellite type per G-SFU:** one sat type at a time. Re-commissioning required to switch. DCU is sat-agnostic (dual filter bank).
5. **SFU pipeline latency: fixed and deterministic.** Not affected by gain, Doppler, beacon, capture, temperature. All cards same HW rev + FW version → identical latency (SYS-LAT-03). DCU uses reported SFU latency for delay compensation offset.
6. **Redundant antennas:** RF combiner/splitter on RF side. No additional matrix ports. SFU internal RF port switch handles T0→T1/T2 and R0→R1/R3. TX+RX switch as a pair.
7. **1 GbE management path:** carries parameter updates, telemetry, alarms, captures, FW transfers. Standard L2 — no TSN/PTP.

### 4.4 Consolidated SFU requirements (from §18, SFU doc)

| Area | Key req IDs | Summary |
|------|-------------|---------|
| Signal processing | SFU-SIG-01 to -07 | MDFT filter bank, all LTE BWs (ASIC; FPGA TBD), Band Gain, Band Doppler, mixed-BW OBG, CW Beacon, FPGA sat native bins |
| RF | SFU-RF-01 to -04 | 1.6 GHz BW, power ranges, hot-standby port switch, 4,096 Msps 14-bit |
| Clock/timing | SFU-CLK-01 to -05 | Primary CDR, secondary 10 MHz, 1PPS for Doppler, inter-SFU coherence, CDR status reporting |
| Latency | SFU-LAT-01 to -04 | Fixed deterministic DL/UL latency, uniform across cards, self-measurement and reporting |
| Management | SFU-MGT-01 to -04 | UTC-Scheduled apply, NV config, zero-touch provisioning, ≤1 s telemetry interval |
| Debug | SFU-DBG-01 to -05 | Static config mode, playback (10 ms LTE frame), capture (multi-trigger), spur detection, spectrum monitor |

---

## 5. ambiguities_gaps_and_risks

### 5.1 Document contradictions / inconsistencies

| # | Issue | Documents | Impact |
|---|-------|-----------|--------|
| 1 | **OI-023 resolution inconsistency.** System doc §7.3 header says "Option B is the selected primary solution" but the decision box below says "Option A (Dual Filter Bank in DCU) is selected." The detailed text and all downstream references (§7.4, §22 SYS-SAT-03, SFU doc §6.3) consistently describe Option A behavior (dual filter bank in DCU, SFU unaffected). **Conclusion: Option A is the actual decision; the §7.3 header line is an editorial error.** | ARCH §7.3 | Low — clear from context, but must be corrected in next doc revision. |
| 2 | **CW Beacon injection point.** SFU doc §6.8 says "after F.B Synthesis and BEFORE Band Doppler and Band Gain." But the DL chain order in §6.5 places Band Gain after F.B. Synthesis and before Band Doppler. If beacon is before both Band Gain and Band Doppler, the chain is: F.B. → Beacon → Band Gain → Band Doppler. §6.8 also says beacon "passes through both Band Doppler and Band Gain processing" — consistent with being before both. **Most likely intent: beacon injected between F.B. Synthesis and Band Gain.** Needs explicit confirmation in a block diagram. | SFU §6.5, §6.8 | Medium — affects beacon power calibration and Band Gain loop behavior. |
| 3 | **Filter bank input frame = 12,288 bins vs. OBG = 5,760 bins.** Each OBG lane carries 5,760 bins. The filter bank sub-band expects 12,288 bins. With 4 OBG lanes total and 2 sub-bands, the mapping is not trivially 1:1. The Bins Selector must aggregate and route bins from multiple OBG lanes to fill each sub-band's 12,288-bin input. The documents do not specify the exact bin-to-lane-to-sub-band mapping rules beyond "NMS configures it." | SFU §6.3, §6.4 | Medium — critical for Bins Selector RTL design. Need to define: can bins from multiple OBG lanes feed the same sub-band? Max bins per sub-band sourced from how many lanes? |
| 4 | **FPGA satellite filter bank BW stated as 1,048.576 MHz in SFU doc §6.4 and §9.1**, but system doc says "TBD (FPGA sat.)" for filter bank native BW. The SFU doc is more specific. **Use SFU doc value.** | ARCH §21.5 vs SFU §6.4 | Low — SFU doc is more recent and explicit. |

### 5.2 TBD / STUB items affecting SFU micro-architecture

| ID | Item | Blocking? | Impact on SFU RTL |
|----|------|-----------|-------------------|
| OI-001 | FPGA satellite bin counts (2.56 MHz slot) | Yes — partial | Bins Selector config tables, filter bank beam slot modes for FPGA sat |
| OI-S01 | SFU DL/UL pipeline latency | No — measured post-synthesis | Latency measurement logic can be designed now; values filled later |
| OI-S02 | Inter-SFU phase skew budget | Partially — affects phase alignment block sizing | Need max skew to size per-lane delay registers |
| OI-S03 | 1PPS electrical spec at ET port | No — handled by board-level IO | Does not affect FPGA logic design directly |
| OI-S04 | QSFP28 breakout cable spec | No | External to FPGA |
| OI-S05 | Playback memory depth, CW Beacon NCO detail | Yes | Drives BRAM/URAM allocation for playback buffer |
| OI-S07 | Management protocol / register map | Yes — blocks SW interface | Must define register map before management logic RTL |
| OI-S08 | Capture spec (points, depth, trigger logic) | Yes | Drives capture buffer sizing and pipeline tap architecture |
| OI-S10 | Spur detection + spectrum monitor resource sharing | Partially | Need to know if one HW block or two |
| OI-S12 | Firmware upgrade procedure | No — handled by boot/config logic | Not on critical DSP path |
| OI-002 | System-level latency budget | No — SFU contributes measured value | |
| OI-004 | QV-band RF centre frequency | No — SFU NCO tunable to 6 GHz at 1 Hz res | Does not affect FPGA architecture |
| OI-005 | Management interface protocol (system level) | Yes — same as OI-S07 | |

### 5.3 Architectural risks

| # | Risk | Severity | Notes |
|---|------|----------|-------|
| R1 | **FPGA satellite ×1.024 clock rate** requires a different PLL configuration and may require different filter bank coefficients. If the same FPGA bitstream must support both modes (switchable at commissioning), the PLL must be runtime-reconfigurable and the filter bank coefficient ROM must hold two sets. If separate bitstreams are acceptable, complexity is lower but FW management doubles. | High | No document states whether one bitstream or two. Must decide. |
| R2 | **Filter bank IP (FLEXFFT240)** is described with specific radix stages and clock rates but no IP vendor or deliverable status is stated. If this is custom RTL, it is on the critical path. If it is a third-party IP, integration risk applies. | High | Need IP sourcing status. |
| R3 | **Bins Selector complexity.** 4 OBG lanes × 5,760 bins/lane = 23,040 input bins. 2 sub-bands × 12,288 bins = 24,576 sub-band bins. The Bins Selector must support arbitrary NMS-configured mapping of any input bin to any sub-band position. If fully flexible, this is a large crossbar. If constrained (e.g., entire OBG lanes map to one sub-band), it simplifies dramatically. No constraint is stated. | Medium | Propose constraining to reduce routing complexity. |
| R4 | **CDC FIFO correction events** must be atomic, timestamped, and reported. The correction mechanism is not defined — is it pointer adjustment? Sample insertion/deletion? This affects latency step behavior and must be defined for SYS-LAT-06 compliance. | Medium | |
| R5 | **Band Doppler NCO compute from TLE.** The SFU must run an orbital mechanics computation (TLE → Doppler) locally. This is non-trivial — typically requires double-precision floating point or fixed-point SGP4/SDP4 propagator. The documents do not specify whether this runs on the RFSoC PS (ARM cores) or PL (FPGA fabric). | Medium | Recommend PS (ARM) for TLE computation, PL for NCO apply. |
| R6 | **Playback block replaces live Aurora input.** If playback is injected pre-Bins Selector, it must generate bin-domain data in the correct OBG frame format. This means the playback memory stores frequency-domain bins, not time-domain IQ. This is not explicitly stated but follows from the injection point. | Low | Confirm with DSP team. |
| R7 | **No specification of OBG Aurora frame format.** Neither document defines the frame structure, header, bin ordering, or framing protocol. Without this, the Aurora Rx/Tx logic and Bins Selector cannot be designed. | High | Likely defined in a separate ICD or to be defined. Must be obtained or authored. |

---

## 6. what_can_be_defined_now

The following SFU blocks and interfaces can be architecturally defined with reasonable confidence from the current documents:

### 6.1 Blocks ready for micro-architecture

| Block | Confidence | Notes |
|-------|------------|-------|
| **Aurora SERDES Rx/Tx wrapper** | Medium | 4-lane Aurora at 19.2 Gbps is well-defined. CDR selection logic (timing master lane) is clear. Blocked on OBG frame format definition. |
| **DL/UL CDC FIFO** | High | Standard clock domain crossing. Correction event detection, timestamping, and reporting logic can be designed. Exact depth depends on clock ratio (known: 341.333 / 320 MHz). |
| **Multi-lane phase alignment** | High | 4 per-lane programmable delay registers. Reference = timing master lane. Compensation depth TBD (OI-S02) but architecture is a simple delay line per lane with NMS-configurable tap. |
| **Band Gain DL/UL** | High | Scalar multiplier per sub-band. −60 to +60 dBFs at 0.1 dBFs → 1,201 steps. Pending register + active register with UTC-Scheduled apply. Clip/round detection post-multiply with alarm counter. |
| **Band Doppler NCO DL/UL** | High | Standard NCO (complex mixer). ±512 Hz at 1 Hz resolution → trivial NCO width. Dual mode: operational (1PPS-triggered from TLE compute) and static (UTC-Scheduled fixed value). |
| **CW Beacon generator** | Medium | NCO + configurable amplitude. Injection point needs final confirmation (see §5.1 item 2). Enable, frequency, power all UTC-Scheduled. |
| **RF port select mux** | High | 3-to-1 mux for TX, 3-to-1 mux for RX. TX+RX switch as pair. UTC-Scheduled. Trivial logic. |
| **DUC / DDC** | High | 4× interpolation / decimation. Standard polyphase or CIC + half-band cascade. Clock rates known. |
| **1PPS input capture** | High | Edge detect on ET port, generate internal UTC tick. Used for Band Doppler apply and UTC-Scheduled parameter activation. |
| **UTC-Scheduled parameter apply engine** | High | Generic mechanism: pending register bank + UTC comparator + 1PPS tick → swap to active + confirmation message to management. Same engine serves all UTC-Scheduled parameters. |
| **Latency measurement** | High | Pipeline sample counter from OBG input to DAC output (DL) and ADC input to OBG output (UL). Report via management. Update on CDC correction events. |
| **Management register file** | Medium | Can define the register address map structure now. Protocol (REST/SNMP/raw register) TBD (OI-S07) but the internal register bank design is independent. |

### 6.2 Blocks that need more information

| Block | Blocker |
|-------|---------|
| **Filter Bank Synthesis / Analysis** | IP sourcing unclear. Coefficient sets for ASIC vs FPGA sat needed. Internal CDC behavior at 341.333/320 MHz boundary needs IP-level spec. |
| **Bins Selector** | OBG frame format undefined. Mapping flexibility (full crossbar vs constrained) not specified. Bin ordering within OBG lane unknown. |
| **Playback** | Memory depth TBD (OI-S05). Need to confirm bin-domain storage format. |
| **Capture** | Capture points, depth, trigger logic all TBD (OI-S08). |
| **Spur Detection / Spectrum Monitor** | Resource sharing question (OI-S10). Update rate, memory, pipeline tap location undefined. |
| **TLE → Doppler compute** | Implementation target (PS vs PL) not specified. SGP4 algorithm precision requirements undefined. |

---

## 7. what_still_needs_clarification

### 7.1 Critical blockers (must resolve before RTL)

| # | Question | Relevant OI | Who decides |
|---|----------|-------------|-------------|
| 1 | **OBG Aurora frame format** — bin ordering, header, framing, error detection, idle pattern. Without this, Aurora Rx/Tx and Bins Selector cannot be coded. | None assigned — gap | Sys Arch + DCU team |
| 2 | **Bins Selector mapping constraints** — is arbitrary bin-to-sub-band routing required, or are OBG lanes pre-assigned to sub-bands? Full flexibility = large mux; constrained = simple. | Not explicitly an OI | Sys Arch |
| 3 | **Single bitstream or dual bitstream for ASIC/FPGA satellite?** This drives PLL architecture, coefficient ROM sizing, and FW management. | Not explicitly an OI | Sys Arch + FW Lead |
| 4 | **Filter bank IP source and delivery schedule.** Custom RTL, third-party IP, or reference design from [7]? | Not explicitly an OI | DSP Lead |
| 5 | **Management register map / API definition.** Needed for management logic RTL and verification. | OI-S07 / OI-005 | FW Lead |
| 6 | **CW Beacon exact injection point in DL chain.** Before or after Band Gain? (See §5.1 item 2.) | Not explicitly an OI | Sys Arch |

### 7.2 Important but not immediately blocking

| # | Question | Relevant OI |
|---|----------|-------------|
| 7 | FPGA satellite 2.56 MHz bin count | OI-001 |
| 8 | Playback memory depth and format | OI-S05 |
| 9 | Capture specification (points, depth, triggers) | OI-S08 |
| 10 | Inter-SFU phase skew budget (drives delay line depth) | OI-S02 |
| 11 | Spur detection vs spectrum monitor — shared HW or separate? | OI-S10 |
| 12 | TLE compute: PS (ARM) or PL? Precision requirements? | Not an OI |
| 13 | CDC FIFO correction mechanism (pointer adjust vs sample insert/delete) | Not an OI |
| 14 | Band Gain loop algorithm — SFU just provides the actuator and Band RMS measurement, but does the loop run on-chip or in Manager SW? | OI-S11 |
| 15 | Non-volatile config implementation — RFSoC on-chip flash, external SPI flash, or host-side? | Not an OI |

---

## 8. first_recommendations

### 8.1 Immediate actions

1. **Define the OBG Aurora frame format** as priority #1. This is the most critical missing specification and blocks Aurora Rx/Tx, Bins Selector, and all integration testing. Draft an ICD covering: frame boundary marker, bin payload format (IQ word width, ordering), per-frame metadata (timestamp, OBG ID, bin count, sat type), idle/error patterns, and CRC/ECC. Coordinate with DCU team.

2. **Resolve Bins Selector mapping constraints** with a written architecture decision. Recommendation: constrain each OBG lane to feed one sub-band only (not split across A and B). This simplifies routing to a 4:2 lane-to-sub-band assignment plus intra-sub-band bin placement. Full flexibility is not justified by any stated use case and creates unnecessary mux complexity.

3. **Decide single vs. dual bitstream** for ASIC/FPGA satellite. Recommendation: single bitstream with runtime-selectable PLL configuration and dual coefficient ROMs. Rationale: simplifies FW management (one image per HW rev) and aligns with the "re-commissioning required" rule (not a hot switch). Verify with RFSoC PLL reconfiguration capability (XPLL dynamic reconfig is supported on UltraScale+).

4. **Confirm CW Beacon injection point** with a formal block diagram review. Recommendation: inject after F.B. Synthesis and before Band Gain DL, so beacon passes through both Band Gain and Band Doppler. This matches the satellite CPBF usage (beacon power and frequency are affected by both corrections, as the satellite sees the composite signal).

### 8.2 Architecture work to start now

5. **Design the UTC-Scheduled parameter apply engine** as a generic, reusable module. It serves all UTC-Scheduled parameters (Band Gain, RF port, CW Beacon, etc.). Define: pending register bank width and depth, UTC comparator, 1PPS edge detector, apply confirmation message format. This block is fully specified and not blocked.

6. **Design the Band Gain block** (DL and UL instances). Scalar multiplier, pending/active register pair, clip detection with alarm counter, round detection. Straightforward and fully specified.

7. **Design the Band Doppler NCO block** (DL and UL instances, sub-band A and B). NCO with operational/static mode select. 1PPS-triggered apply. Define interface to TLE compute module (Doppler value in, NCO frequency word out). NCO word width is trivially small (±512 Hz at 1 Hz = 10-bit signed).

8. **Prototype the multi-lane phase alignment block.** Per-lane programmable delay using shift registers or BRAM-based delay lines. Size for a conservative maximum skew estimate (e.g., 100 ns → ~34 samples at 341 MHz) until OI-S02 is resolved.

9. **Allocate TLE → Doppler computation to the RFSoC PS (ARM Cortex-A53).** SGP4/SDP4 propagation with double-precision math is a natural fit for the ARM cores. The PS computes Doppler values and writes them to PL registers via AXI. The PL applies at the next 1PPS tick. This cleanly separates orbital mechanics (SW, ~1 Hz update) from real-time NCO control (HW, sample-rate).

### 8.3 Verification and debug

10. **Define a latency measurement RTL block early.** Insert a frame counter at OBG input and a corresponding counter at DAC output (DL) / ADC input to OBG output (UL). The difference is the pipeline latency. Report via management register. This is a low-cost block that provides immediate value during bring-up.

11. **Plan for the Playback block to store bin-domain data**, not time-domain IQ. The injection point is pre-Bins Selector (OBG bin domain). The playback memory must hold at least 10 ms of bin data at the target BW. For a single 5 MHz cell (60 bins × 83.333 kHz sample rate × 10 ms) this is modest; for full SFU capacity it may require significant BRAM/URAM. Size conservatively and defer exact depth to OI-S05 resolution, but reserve URAM budget now.

12. **Establish a register map skeleton.** Even with OI-S07 open (protocol TBD), the internal register address space can be defined: per-sub-band Band Gain, per-sub-band Band Doppler, RF port select, CW Beacon params, AGB config, clock mode, static mode, phase alignment delays, latency readback, telemetry (RMS, temperature, CDR status, Aurora lane status, clip counters, FW version). The register bank is protocol-agnostic — the protocol adapter sits on top.

### 8.4 Risk mitigations

13. **Obtain the filter bank IP delivery schedule and integration guide** from DSP Lead immediately. If the IP is not yet available, begin designing the Bins Selector and Band Gain/Doppler blocks independently — they interface to the filter bank via well-defined bin buses and can be verified in isolation.

14. **Request the DCU team's OBG output format specification** or, if none exists, jointly author one. The OBG ICD is the single most important interface document for SFU RTL work.

---

*End of review note.*
