# Change-set — FAD batch 04 (Chapter 2.3 review)

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Scope:** FAD §2.3 review. Touches §1.4.1, §2.1, §4.5, §5, §6 as *ripple edits forced by* ADR-0012 (SmartConnect) and ADR-0013 (`tle_compute` fold) — NOT a §4/§5/§6 review. Plus two new ADRs.
**Apply method:** `str_replace` exact find-replace. Preserve `→ × ± ~ § ↔` glyphs and backticks.
**Pre-flight:** apply after batches 01–03. Two ADR files are delivered alongside; commit them under `arch/adr/`.

New ADRs (commit first): `arch/adr/0012-management-csr-fabric-ip.md`, `arch/adr/0013-fold-tle-compute-into-band-doppler.md` (both `status: proposed`).
Diagram: replace `arch/fad/diagrams/infrastructure.drawio` with the delivered file; re-render `infrastructure.png`.

Groups: **A** ADR-0012/SmartConnect · **B** ADR-0013/`tle_compute` fold · **C** P1 ADR-0004 proposed · **D** P2 filter-bank · **E** P3 static-mode Doppler · **F** P4 1PPS coherence · **G** D5 platform/context exception. Suggested commits: A+C+D+E+F+G as `docs(fad): chapter-2.3 review fixes`; B as `docs(fad): fold tle_compute into band_doppler (ADR-0013)`.

**Carried, NOT fixed here:** FAD-ARCH-01 still states "PL hosts AXI bridge (`mgmt_if`), register fabric (`reg_bus`)", which contradicts FAD-ARCH-10 + ADR-0012 (no SFU-owned CSR fabric block). This is the parked R4 item; it needs a §6-inventory decision (do `reg_bus`/`mgmt_if` rows exist?) and is left for its own pass.

---

## GROUP A — ADR-0012: AXI SmartConnect (D1 blocker)

### A1 — FAD-ARCH-10 → SmartConnect  ·  `arch/fad/FAD.md`

**FIND:**
```
| FAD-ARCH-10 | CSR fabric uses the Xilinx **AXI4-Lite Interconnect** IP single-clocked at `pl_clk0`. No SFU-owned bridge or CSR fabric block. Address decode is the Interconnect IP's responsibility; per-block CSR↔core CDC is internal to each block, captured in each MS against the approved mechanism set in FAD §4.5.1. | N/A — see §2.3 |
```
**REPLACE:**
```
| FAD-ARCH-10 | CSR fabric uses the AMD **AXI SmartConnect** IP single-clocked at `pl_clk0` ([ADR-0012](../adr/0012-management-csr-fabric-ip.md); the classic AXI Interconnect IP is discontinued). Two SI masters (PS `M_AXI_HPM0_FPD`, `nv_cfg` POR-restore master); one AXI4-Lite MI slave per block. No SFU-owned bridge or CSR fabric block. Address decode is the SmartConnect's responsibility; per-block CSR↔core CDC is internal to each block, captured in each MS against the approved mechanism set in FAD §4.5.1. | [ADR-0012](../adr/0012-management-csr-fabric-ip.md); see §2.3 |
```

### A2 — §2.3 management-entry prose: IP + PS-port precision  ·  `arch/fad/FAD.md`

**FIND:**
```
Management traffic enters the SFU at the platform 1 GbE RJ-45 port (via TeraBox rear panel). The Ethernet GEM, management protocol stack, and TLE/SGP4 propagation run in PS software (out of scope per FAD-ARCH-01). The PS exposes a single AXI4 master to PL through `M_AXI_HPM0_FPD`, clocked by `pl_clk0` (= `clk_mgmt`, 100 MHz). A Xilinx **AXI4-Lite Interconnect IP** in `clk_mgmt` fans out to one S_AXI-Lite slave port per infrastructure and datapath block — no SFU-owned bridge or CSR fabric block between them. Address decode is the Interconnect IP's job; each block's address range is fixed in the IP Integrator block design and recorded in the `mgmt_regmap` ICD.
```
**REPLACE:**
```
Management traffic enters the SFU at the platform 1 GbE RJ-45 port (via TeraBox rear panel). The Ethernet GEM, management protocol stack, and TLE/SGP4 propagation run in PS software (out of scope per FAD-ARCH-01). The PS exposes a single AXI4 master to PL through `M_AXI_HPM0_FPD` (32-bit data width is sufficient for CSR access), clocked by `pl_clk0` (= `clk_mgmt`, 100 MHz). An AMD **AXI SmartConnect IP** ([ADR-0012](../adr/0012-management-csr-fabric-ip.md)) in `clk_mgmt` fans out to one AXI4-Lite S_AXI slave port per infrastructure and datapath block — no SFU-owned bridge or CSR fabric block between them. The SmartConnect performs the AXI4→AXI4-Lite conversion internally, so the AXI4-Lite subset begins inside the fabric, not at the PS port. Address decode is the SmartConnect's job; each block's address range is fixed in the IP Integrator block design and recorded in the `mgmt_regmap` ICD.
```

### A3 — §2.3 `nv_cfg` second-master sentence  ·  `arch/fad/FAD.md`

**FIND:**
```
`nv_cfg` is a second AXI master on the same Interconnect: on POR it pushes the saved CSR values back into the register files of all controlled blocks before the SFU enters service. Detailed restore sequence is in §4.4.4 and the `nv_cfg` MS.
```
**REPLACE:**
```
`nv_cfg` is a second AXI master on the same SmartConnect: on POR it pushes the saved CSR values back into the register files of all controlled blocks before the SFU enters service. Detailed restore sequence is in §4.4.4 and the `nv_cfg` MS.
```

### A4 — §2.3 pre-load step "Interconnect" mention  ·  `arch/fad/FAD.md`

**FIND:**
```
PS sw writes both fields through the AXI4-Lite Interconnect into the target block's *shadow* register (the live register is untouched).
```
**REPLACE:**
```
PS sw writes both fields through the SmartConnect into the target block's *shadow* register (the live register is untouched).
```

### A5 — §2.3 nv_cfg restore "Interconnect" mentions  ·  `arch/fad/FAD.md`

(See E1 — this sentence is also edited by Group E for the Doppler-restore qualifier. Apply **E1**, which contains both the SmartConnect and static-mode-Doppler changes. If applying A and E separately, do A5 first, then E1's FIND must be updated to the A5 output.)

---

## GROUP B — ADR-0013: fold `tle_compute` into `band_doppler`

### B1 — FAD-ARCH-01: tle_compute clause only  ·  `arch/fad/FAD.md`

Edits only the NCO-load clause. The `mgmt_if`/`reg_bus` wording is intentionally left (parked R4).

**FIND:**
```
| FAD-ARCH-01 | PS software out of scope of this repo; PS hosts Ethernet GEM, mgmt protocol stack, and TLE/SGP4 propagation. PL hosts AXI bridge (`mgmt_if`), register fabric (`reg_bus`), and 1PPS-edge NCO load (`tle_compute` PL portion). | [TBD: ADR-0003 PS-PL partitioning, owner: Marco] |
```
**REPLACE:**
```
| FAD-ARCH-01 | PS software out of scope of this repo; PS hosts Ethernet GEM, mgmt protocol stack, and TLE/SGP4 propagation. PL hosts AXI bridge (`mgmt_if`), register fabric (`reg_bus`), and 1PPS-aligned Band Doppler NCO load + interpolation (in `band_doppler`, per [ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)). | [TBD: ADR-0003 PS-PL partitioning, owner: Marco] |
```
*(If batch 01 did not set the owner to Marco, the trailing cell reads `owner: TBD` — adjust the FIND accordingly.)*

### B2 — FAD-ARCH-11 mechanism (b)  ·  `arch/fad/FAD.md`

**FIND:**
```
(b) 1PPS-aligned apply for operational Band Doppler — PS sw writes the next NCO word ~1 s in advance; `tle_compute` (PL) latches at the next 1PPS edge.
```
**REPLACE:**
```
(b) 1PPS-aligned apply for operational Band Doppler — PS sw writes the next Doppler data ~1 s in advance; `band_doppler` (PL) latches and interpolates the NCO word at the 1PPS edge ([ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)).
```

### B3 — §2.1 Band Doppler narrative  ·  `arch/fad/FAD.md`

**FIND:**
```
`band_doppler_dl` applies Band Doppler correction at sub-band granularity. In operational mode, the correction is Local Autonomous: the SFU computes the NCO frequency word from TLE ephemeris data (supplied by Manager SW via `tle_compute`) and applies it at the 1PPS tick. The Manager SW does not write Doppler values directly during live tracking. In static/debug mode, the NCO is held at a fixed configured value (UTC-scheduled). Range: ±512 Hz (ASIC satellite) / ±524.288 Hz (FPGA satellite), 1 Hz resolution. (SFU-001 §6.6; ARCH-001 §15.3)
```
**REPLACE:**
```
`band_doppler_dl` applies Band Doppler correction at sub-band granularity. In operational mode, the correction is Local Autonomous: PS software computes per-second Doppler data from TLE ephemeris (SGP4, out of scope per FAD-ARCH-01) and writes it ~1 s ahead over S_AXI-Lite; `band_doppler` latches and interpolates the NCO frequency word and applies it from the 1PPS tick ([ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)). Manager SW does not write Doppler values directly during live tracking. In static/debug mode, the NCO is held at a fixed configured value (UTC-scheduled). Range: ±512 Hz (ASIC satellite) / ±524.288 Hz (FPGA satellite), 1 Hz resolution. (SFU-001 §6.6; ARCH-001 §15.3)
```

### B4 — §2.3 "Band Doppler 1PPS-aligned apply path" subsection rewrite  ·  `arch/fad/FAD.md`

Replaces the three bullet points + the two paragraphs that follow them, through the "Synchronisation" paragraph.

**FIND:**
```
- **PS sw** runs SGP4 propagation on the supplied TLE data and computes the next NCO frequency word for each Band Doppler instance once per second.
- **PS sw writes the new NCO word ~1 second ahead of when it must take effect**, through the AXI4-Lite Interconnect, into a "next" register at `tle_compute` (PL portion).
- **PL** latches "next" → live NCO output to `band_doppler_dl/ul` on the synchronised 1PPS tick from `pps_capture`.

This is structurally the same shadow-active commit as UTC-scheduled apply (Manager SW upstream, PS sw doing the pre-load via AXI, PL doing the commit), but with a different trigger: the apply moment is **the 1PPS edge itself**, not a UTC-counter match. SFU-001 calls this category *Local Autonomous* because the SFU applies without per-update command latency from outside the card — but the apply moment is still UTC-derived, just with hardware-precise alignment (the 1PPS edge has sub-µs accuracy at the ET port, whereas a UTC-counter match resolves at the `clk_mgmt` period).

**Why this mechanism rather than UTC-Scheduled?** Band Doppler updates every second; the per-update jitter and granularity of the UTC-counter compare path are too coarse. The 1PPS hardware edge gives the precision required for continuous Doppler tracking without an in-band coordination channel.

**Synchronisation between `tle_compute` and `band_doppler`.** Both blocks consume the **same synchronised 1PPS tick** produced by `pps_capture` in `clk_dsp`. `tle_compute` uses it as the latch enable for `next → live`; `band_doppler` uses it as the apply boundary for the NCO frequency change. Because both edges come from the same single-source synchroniser inside `pps_capture`, the NCO update and the `band_doppler` consumption of the new value are coherent within one `clk_dsp` cycle. No additional inter-block handshake is needed.
```
**REPLACE:**
```
- **PS sw** runs SGP4 propagation on the supplied TLE data and computes the next Band Doppler data for each instance once per second.
- **PS sw writes the new Doppler data ~1 second ahead of when it must take effect**, through the SmartConnect, into a "next" register inside the target `band_doppler` instance.
- **`band_doppler` (PL)** latches "next" → live and interpolates the NCO frequency across the second, updating every `clk_dsp` cycle, gated from the synchronised 1PPS tick produced by `pps_capture` (per [ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)).

This is structurally the same shadow-active commit as UTC-scheduled apply (Manager SW upstream, PS sw doing the pre-load via AXI, PL doing the commit), but with a different trigger: the apply moment is **the 1PPS edge itself**, not a UTC-counter match. SFU-001 calls this category *Local Autonomous* because the SFU applies without per-update command latency from outside the card — but the apply moment is still UTC-derived, just with hardware-precise alignment (the 1PPS edge has sub-µs accuracy at the ET port, whereas a UTC-counter match resolves at the `clk_mgmt` period).

Over a 1 s update interval, the Doppler trajectory is locally linear to within the 1 Hz NCO resolution (the interpolation error scales with Doppler acceleration × T² and is second-order over 1 s at this band-Doppler scale); across a full pass it is the usual S-curve, steepest at TCA, so the interpolation is a per-second local fit rather than a global linear assumption. `[INFERRED: the exact intra-second law (interpolate-between-endpoints vs extrapolate-with-rate) and the PS↔PL data contract are `band_doppler` MS + reference-model scope; expiry: `band_doppler` MS design_ready.]`

**Why this mechanism rather than UTC-Scheduled?** Band Doppler updates every second; the per-update jitter and granularity of the UTC-counter compare path are too coarse. The 1PPS hardware edge gives the precision required for continuous Doppler tracking without an in-band coordination channel.

**1PPS coherence (intra-block).** The latch of "next" → live and the NCO frequency change occur inside the same `band_doppler` instance, both gated by the single synchronised 1PPS pulse delivered by `pps_capture` over CDC-11 (`clk_dsp`). Because there is no inter-block boundary, the new value is consumed on the same `clk_dsp` cycle it is applied by construction — no cross-block handshake exists or is required.
```

### B5 — §2.3 apply table footnote cleanup  (none needed — table does not name `tle_compute`)

### B6 — §4.5 CDC-08 owner/protocol  ·  `arch/fad/FAD.md`

**FIND:**
```
| CDC-08 | mgmt | dsp | TLE-derived NCO frequency words (Band Doppler) | Shadow-active atomic at 1PPS — protocol in `tle_compute` + `band_doppler` MSs | `tle_compute` |
```
**REPLACE:**
```
| CDC-08 | mgmt | dsp | Band Doppler data (PS-supplied, per-second) | Shadow-active atomic at 1PPS — protocol in `band_doppler` MS | `band_doppler` |
```

### B7 — §4.5 CDC-11 consumer  ·  `arch/fad/FAD.md`

**FIND:**
```
3. Two-flop synchroniser into `clk_dsp` (CDC-11) — drives `tle_compute` latch enable and any other dsp-domain 1PPS consumers.
```
**REPLACE:**
```
3. Two-flop synchroniser into `clk_dsp` (CDC-11) — drives the `band_doppler` 1PPS latch/interpolation and any other dsp-domain 1PPS consumers.
```

### B8 — §2.3 event-log link sentence  ·  `arch/fad/FAD.md`

**FIND:**
```
Apply confirms and apply misses are the link from `utc_apply_engine` and `tle_compute` back into the log, giving Manager SW the upstream signal it needs to detect failures per ARCH-001 §15.4.
```
**REPLACE:**
```
Apply confirms and apply misses are the link from `utc_apply_engine` and `band_doppler` back into the log, giving Manager SW the upstream signal it needs to detect failures per ARCH-001 §15.4.
```

### B9 — §5 budget negligible-consumer list  ·  `arch/fad/FAD.md`

**FIND:**
```
| Negligible consumers | `band_gain` (×2), `rf_port_sel`, `mgmt_if`, `reg_bus`, `utc_apply_engine`, `tle_compute`, `pps_capture`, `clock_ctrl` | Distributed only | < 1 BRAM each | CSRs, counters, shadow registers, small FSM state. No BRAM/URAM allocation expected. |
```
**REPLACE:**
```
| Negligible consumers | `band_gain` (×2), `rf_port_sel`, `mgmt_if`, `reg_bus`, `utc_apply_engine`, `pps_capture`, `clock_ctrl` | Distributed only | < 1 BRAM each | CSRs, counters, shadow registers, small FSM state. No BRAM/URAM allocation expected. |
```

### B10 — §6 note 6.2.e (utc_apply_engine)  ·  `arch/fad/FAD.md`

**FIND:**
```
The `utc_apply_bus` between this block and controlled endpoints is an ICD candidate that must be frozen before the first UTC-scheduled-parameter MS is authored. Operational-mode `band_doppler` updates do **not** route through this engine: `tle_compute` writes directly to the live NCO at 1PPS using the Local Autonomous mechanism.
```
**REPLACE:**
```
The `utc_apply_bus` between this block and controlled endpoints is an ICD candidate that must be frozen before the first UTC-scheduled-parameter MS is authored. Operational-mode `band_doppler` updates do **not** route through this engine: `band_doppler` latches and interpolates the live NCO at 1PPS using the Local Autonomous mechanism ([ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)).
```

### B11 — §6 row 18 delete + row 8 note + total  ·  `arch/fad/FAD.md`

Delete the `tle_compute` row entirely:

**FIND:**
```
| 18 | [`tle_compute`](../modules/tle_compute.md) | new | `mgmt`, `dsp` | SFU-001 §6.6, §7.2, §11.3 | draft | 1 PL-side Doppler load/apply-control block; PS-owned SGP4/orbital propagation is out of scope per FAD-ARCH-01 |
```
**REPLACE:** *(empty — remove the line, including its trailing newline)*

Extend the `band_doppler` row note:

**FIND:**
```
| 8 | [`band_doppler`](../modules/band_doppler.md) | new | `dsp`, `mgmt` | SFU-001 §6.6, §7.2 | draft | 2 instances: `band_doppler_dl`, `band_doppler_ul` |
```
**REPLACE:**
```
| 8 | [`band_doppler`](../modules/band_doppler.md) | new | `dsp`, `mgmt` | SFU-001 §6.6, §7.2, §11.3 | draft | 2 instances: `band_doppler_dl`, `band_doppler_ul`. Owns the 1PPS-aligned NCO load + intra-second interpolation absorbed from the former `tle_compute` (ADR-0013); PS-owned SGP4/orbital propagation out of scope (FAD-ARCH-01) — see note 6.2.j |
```

Decrement the total:

**FIND:**
```
**Total:** 23 module-spec rows. The corresponding top-level RTL may contain more than 23 instances because some module specs are reused for DL and UL instances.
```
**REPLACE:**
```
**Total:** 22 module-spec rows. The corresponding top-level RTL may contain more than 22 instances because some module specs are reused for DL and UL instances. `[TBD: reconcile stated total against visible rows and the missing row-16 index — pre-existing §6.1 bookkeeping discrepancy, owner: Marco]`
```

### B12 — §6 note 6.2.j repurpose  ·  `arch/fad/FAD.md`

**FIND:**
```
**j. `tle_compute` naming caveat.**

`tle_compute` is retained to preserve current FAD naming, but the MS must constrain the scope to the PL-side Doppler load/apply function. SGP4/orbital propagation, Doppler-word generation policy, and management protocol handling remain PS software / Manager SW responsibilities unless a later PS/PL-partition ADR changes that boundary.
```
**REPLACE:**
```
**j. `band_doppler` absorbs the former `tle_compute` (ADR-0013).**

Operational Band Doppler NCO load + intra-second interpolation is owned by `band_doppler`, not a separate block (ADR-0013 folded the former `tle_compute`). The `band_doppler` MS constrains PL scope to the per-direction Doppler load/apply/interpolation function. SGP4/orbital propagation, Doppler-data generation policy, and management protocol handling remain PS software / Manager SW responsibilities unless a later PS/PL-partition ADR (ADR-0003) changes that boundary.
```

---

## GROUP C — P1: ADR-0004 is proposed, not settled

### C1 — §2.3 ADR-0004 reference  ·  `arch/fad/FAD.md`

**FIND:**
```
Engine architecture, miss-handling policy, miss-alarm taxonomy, and the `utc_apply_bus` ICD shape are captured in [ADR-0004](../adr/0004-utc-apply-engine.md).
```
**REPLACE:**
```
Engine architecture, miss-handling policy, miss-alarm taxonomy, and the `utc_apply_bus` ICD shape are proposed in [ADR-0004](../adr/0004-utc-apply-engine.md) (status: proposed; acceptance gated per FAD-ARCH-03 before the first UTC-scheduled-parameter MS is authored).
```

---

## GROUP D — P2: filter-bank apply item (SFU-local, not DCU bank-select)

### D1 — §2.3 apply-list parameters  ·  `arch/fad/FAD.md`

**FIND:**
```
Parameters using this mechanism in the SFU: Band Gain DL/UL, RF port selection, CW beacon (enable/freq/power), `bins_sel` routing tables, filter-bank configuration, clock mode, static-config-mode, and the static-mode `band_doppler` value (commissioning/debug only). The complete parameter-by-parameter inventory is in SFU-001 §17.1.
```
**REPLACE:**
```
Parameters using this mechanism in the SFU: Band Gain DL/UL, RF port selection, CW beacon (enable/freq/power), `bins_sel` routing tables, `filter_bank_*` wrapper configuration (SFU-local — **not** the DCU per-OBG ASIC/FPGA bank select, which is DCU-owned per INV-08; parameters per the `filter_bank_*` MS), clock mode, static-config-mode, and the static-mode `band_doppler` value (commissioning/debug only). The complete parameter-by-parameter inventory is in SFU-001 §17.1.
```

### D2 — §2.3 apply table row label  ·  `arch/fad/FAD.md`

**FIND:**
```
| UTC-Scheduled | Low-rate coordinated config (Band Gain, RF port, CW beacon, `bins_sel`, filter-bank, clock mode, static-mode Doppler) | UTC counter == configured timestamp | `clk_mgmt` cycle (~10 ns) |
```
**REPLACE:**
```
| UTC-Scheduled | Low-rate coordinated config (Band Gain, RF port, CW beacon, `bins_sel`, `filter_bank_*` wrapper cfg, clock mode, static-mode Doppler) | UTC counter == configured timestamp | `clk_mgmt` cycle (~10 ns) |
```

---

## GROUP E — P3: static-mode Doppler in nv_cfg restore  (+ A5 SmartConnect mentions)

### E1 — §2.3 nv_cfg restore sentence  ·  `arch/fad/FAD.md`

Combines the P3 Doppler qualifier and the two Group-A "Interconnect"→"SmartConnect" mentions in this sentence.

**FIND:**
```
`nv_cfg` handles non-volatile configuration persistence and power-on restore. On POR, after `clk_mgmt` and the AXI4-Lite Interconnect are alive, `nv_cfg` becomes a transient AXI master and writes the saved CSR values back into every controlled block through the same Interconnect — restoring the device to its last-commissioned state without NMS intervention. Persisted parameters include clock mode, timing-master OBG lane, beam-slot BW modes, Band Gain values, Band Doppler offsets, RF port selection, CW beacon config, static-config-mode state, and G-SFU group assignment. `[INFERRED from SFU-001 §17.2 — exact restore sequencing, POR FSM, and flash protocol are MS / uArch scope.]`
```
**REPLACE:**
```
`nv_cfg` handles non-volatile configuration persistence and power-on restore. On POR, after `clk_mgmt` and the AXI SmartConnect are alive, `nv_cfg` becomes a transient AXI master and writes the saved CSR values back into every controlled block through the same SmartConnect — restoring the device to its last-commissioned state without NMS intervention. Persisted parameters include clock mode, timing-master OBG lane, beam-slot BW modes, Band Gain values, static-mode Band Doppler offsets (operational next/live NCO words are **not** restored from NVM — they are recomputed from TLE by PS sw and reloaded at 1PPS), RF port selection, CW beacon config, static-config-mode state, and G-SFU group assignment. `[INFERRED from SFU-001 §17.2 — exact restore sequencing, POR FSM, and flash protocol are MS / uArch scope.]`
```

---

## GROUP F — P4: 1PPS coherence contract + clock_ctrl dangling ref

### F1 — §2.3 "Timing and clock plane" coherence claim  ·  `arch/fad/FAD.md`

Replaces the overclaim with a bounded-but-independent contract; the hard coherence requirement is now intra-block (B4).

**FIND:**
```
Both synchronised pulses originate from the same input flop chain, so the `clk_mgmt` UTC counter and the `clk_dsp` Doppler load remain aligned to the same physical 1PPS edge (subject to the per-domain synchroniser latency, which is deterministic and accounted for in §4.5).
```
**REPLACE:**
```
Both synchronised pulses derive from the same glitch-filtered 1PPS edge, but CDC-10 (`clk_mgmt`) and CDC-11 (`clk_dsp`) are **independent** two-flop synchronisers: each adds a bounded capture latency (~2 destination-clock cycles + up to one cycle of edge-phase uncertainty) with no fixed cross-domain cycle relationship — none is assumed and none is needed, since the 1-second update budget dwarfs the few-cycle inter-domain spread. The one hard coherence requirement is intra-domain: within `clk_dsp`, the single CDC-11 pulse is the common latch/apply boundary for `band_doppler` (see "Band Doppler 1PPS-aligned apply path" above). The per-domain capture latency is deterministic and accounted for in §4.5.
```

### F2 — §2.3 clock_ctrl dangling fragment  ·  `arch/fad/FAD.md`

**FIND:**
```
`clock_ctrl` (see §4.2.3) 

The dotted clock-source lines in the diagram are intentionally high-level.
```
**REPLACE:**
```
`clock_ctrl` is the PL-side clock supervisor (primary/secondary source selection, lock-status read-back, sat-switch sequencing trigger, reset gating). PS software owns the actual clock-device writes (Si5394, LMK04821, LMX2594); the PS/PL contract, lock-status path, and failure reporting are in §4.6 and the `clock_ctrl` MS.

The dotted clock-source lines in the diagram are intentionally high-level.
```

---

## GROUP G — D5: platform/context exception class

### G1 — §2 rule (top of §2)  ·  `arch/fad/FAD.md`

**FIND:**
```
The diagrams in §2 are functional-block views. The canonical module-spec inventory is FAD §6.1. Every functional block shown in §2 shall have a corresponding §6.1 inventory row, except for explicitly labelled visual grouping containers such as `DL sub-band proc ×2` and `UL sub-band proc ×2`. Those containers are diagram-only and do not create module-spec rows.
```
**REPLACE:**
```
The diagrams in §2 are functional-block views. The canonical module-spec inventory is FAD §6.1. Every SFU-owned PL functional block shown in §2 shall have a corresponding §6.1 inventory row, with two exception classes that are explicitly labelled and do not create module-spec rows: (1) visual grouping containers such as `DL sub-band proc ×2` and `UL sub-band proc ×2`; (2) platform / context blocks shown for orientation — PS software (Ethernet GEM, mgmt stack, SGP4), AMD infrastructure IP instantiated in the block design (AXI SmartConnect, `proc_sys_reset`), and external interfaces/sources (QSFP28, SSMC, 1PPS/10 MHz inputs). Context blocks are rendered faded or in the PS/Xilinx-IP legend colour.
```

### G2 — §6.1 inventory rule  ·  `arch/fad/FAD.md`

**FIND:**
```
Every functional block shown in §2 shall have a corresponding row here, except for explicitly labelled visual grouping containers such as `DL sub-band proc ×2` and `UL sub-band proc ×2`. Those containers are diagram-only and do not create module-spec rows.
```
**REPLACE:**
```
Every SFU-owned PL functional block shown in §2 shall have a corresponding row here, except for the two explicitly labelled exception classes defined in §2: (1) visual grouping containers (`DL sub-band proc ×2`, `UL sub-band proc ×2`); (2) platform / context blocks (PS software, AMD infrastructure IP such as the AXI SmartConnect and `proc_sys_reset`, and external interfaces). Those are diagram-only and do not create module-spec rows.
```

### G3 — Figure 2.3 caption  ·  `arch/fad/FAD.md`

**FIND:**
```
*Figure 2.3 — SFU infrastructure and shared view. Management plane (left), timing/clock plane (right), controlled datapath endpoints (bottom, faded). Source: `arch/fad/diagrams/infrastructure.drawio`.*
```
**REPLACE:**
```
*Figure 2.3 — SFU infrastructure and shared view. Management plane (left), timing/clock plane (right), controlled datapath endpoints (bottom, faded). PS-software and AMD-IP blocks (Ethernet GEM, mgmt stack, SGP4, AXI SmartConnect, `proc_sys_reset`) are platform/context, shown for orientation only and not §6 rows (§2 exception class 2). Source: `arch/fad/diagrams/infrastructure.drawio`.*
```

---

## GROUP H — OPTIONAL housekeeping change-log row  ·  `arch/fad/FAD.md`

```
| 0.7.4 | 2026-05-28 | Marco Pausini | **Chapter-2.3 review fixes (pre-G1).** CSR fabric pinned to AMD AXI SmartConnect (ADR-0012; classic AXI Interconnect discontinued) — FAD-ARCH-10, §2.3 prose, diagram unified; PS `M_AXI_HPM0_FPD` corrected to AXI4 (Lite conversion in fabric). `tle_compute` folded into `band_doppler` (ADR-0013): §6 row 18 removed (total 23→22, pre-existing count discrepancy flagged), FAD-ARCH-01/-11, §2.1, §2.3 apply path, §4.5 CDC-08/CDC-11, §5 budget, notes 6.2.e/j updated. ADR-0004 reference marked proposed/gated (P1). Filter-bank apply item scoped to SFU-local `filter_bank_*` wrapper cfg, disowning DCU bank-select per INV-08 (P2). nv_cfg restore qualified to static-mode Band Doppler only (P3). 1PPS coherence rewritten as bounded-independent CDC-10/11 + intra-block band_doppler latch (P4); clock_ctrl dangling ref fixed. §2/§6 gained a platform/context exception class for PS-sw and AMD-IP blocks (D5). FAD-ARCH-01 mgmt_if/reg_bus vs FAD-ARCH-10/ADR-0012 contradiction (R4) flagged, deferred to §6-inventory pass. infrastructure.drawio updated + PNG re-rendered; ADR-0012, ADR-0013 added (proposed). |
```

---

## Post-apply check

```sh
# Group A — SmartConnect
grep -c "AXI4-Lite Interconnect\|same Interconnect\|the Interconnect" arch/fad/FAD.md   # expect 0
grep -n "AXI SmartConnect" arch/fad/FAD.md   # FAD-ARCH-10, §2.3 ×3
# Group B — tle_compute fold
grep -c "tle_compute" arch/fad/FAD.md        # expect 0 (changelog rows for 0.4/0.7.4 mention it as history — acceptable; verify only §1–§6 body is clean)
grep -n "absorbs the former" arch/fad/FAD.md # note 6.2.j
grep -n "Total:..22 module-spec" arch/fad/FAD.md
grep -c "CDC-08 | mgmt | dsp | Band Doppler data" arch/fad/FAD.md  # 1
# Groups C–G
grep -n "status: proposed; acceptance gated" arch/fad/FAD.md       # C1
grep -n "filter_bank_\* wrapper configuration (SFU-local" arch/fad/FAD.md   # D1
grep -n "operational next/live NCO words are \*\*not\*\* restored" arch/fad/FAD.md  # E1
grep -n "are \*\*independent\*\* two-flop synchronisers" arch/fad/FAD.md    # F1
grep -n "PL-side clock supervisor" arch/fad/FAD.md                 # F2
grep -n "platform / context blocks shown for orientation" arch/fad/FAD.md  # G1
# diagram
grep -c "tle_compute\|AXI4-Lite · pl_clk0\|M+S on Interconnect" arch/fad/diagrams/infrastructure.drawio  # expect 0
# ADRs present
ls arch/adr/0012-management-csr-fabric-ip.md arch/adr/0013-fold-tle-compute-into-band-doppler.md
```

Render §2.3 + `infrastructure.png` and eyeball: SmartConnect named consistently, no `tle_compute` block, `band_doppler` endpoint shows "1PPS NCO load + interpolation", coherence paragraph no longer claims cross-domain alignment.
