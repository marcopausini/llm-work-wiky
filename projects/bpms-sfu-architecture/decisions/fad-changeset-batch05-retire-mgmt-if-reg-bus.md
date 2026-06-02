# Change-set — FAD batch 05 (retire `mgmt_if` / `reg_bus` block nouns + DDR4 ref cleanup)

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Scope:** Retire `mgmt_if` and `reg_bus` as SFU **block nouns** (R4 root-cause clean, forced into the open by ADR-0012). Keep the register-bus *convention* and the `register_bus` ICD. Plus DDR4 reference fixes (ADR-0010 misref; §5.4 path sentence → deferred marker).
**Apply method:** `str_replace` exact find-replace. Preserve `→ × ± § ↔ ²` glyphs and backticks.
**Pre-flight:** **batch 04 is applied.** Two hunks (A1 FAD-ARCH-01, A8 §5 budget) are written against the **post-batch-04** text — their FIND strings include the batch-04 edits.

What dies: `mgmt_if` (AXI-bridge block), `reg_bus` (register-fabric block) — both contradict FAD-ARCH-10 + ADR-0012 (SmartConnect + per-block Lite slaves).
What stays: "register bus" as a *convention*, the `register_bus` ICD, §11.2 section (reworded), "Management fabric" generic phrasing, CDC-06/07 (owner reframed per-block).
Flagged, not fixed: possible `register_bus` vs `mgmt_regmap` ICD redundancy → ICD pass.

Groups: **A** block-noun retirement · **B** DDR4/ADR-0010 cleanup. Suggested commits: A as `docs(fad): retire mgmt_if/reg_bus block nouns (ADR-0012 cleanup)`; B as `docs(fad): fix PL-DDR4 ADR ref + defer capture/playback path`.

---

## GROUP A — retire `mgmt_if` / `reg_bus` block nouns

### A1 — FAD-ARCH-01 (POST-batch-04 anchor)  ·  `arch/fad/FAD.md`

**FIND:**
```
| FAD-ARCH-01 | PS software out of scope of this repo; PS hosts Ethernet GEM, mgmt protocol stack, and TLE/SGP4 propagation. PL hosts AXI bridge (`mgmt_if`), register fabric (`reg_bus`), and 1PPS-aligned Band Doppler NCO load + interpolation (in `band_doppler`, per [ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)). | [TBD: ADR-0003 PS-PL partitioning, owner: Marco] |
```
**REPLACE:**
```
| FAD-ARCH-01 | PS software out of scope of this repo; PS hosts Ethernet GEM, mgmt protocol stack, and TLE/SGP4 propagation. PL hosts per-block AXI4-Lite CSR slaves behind the AMD AXI SmartConnect IP ([ADR-0012](../adr/0012-management-csr-fabric-ip.md); no SFU-owned bridge or CSR-fabric block), and 1PPS-aligned Band Doppler NCO load + interpolation (in `band_doppler`, per [ADR-0013](../adr/0013-fold-tle-compute-into-band-doppler.md)). | [TBD: ADR-0003 PS-PL partitioning, owner: Marco] |
```

### A2 — FAD-ARCH-06  ·  `arch/fad/FAD.md`

**FIND:**
```
| FAD-ARCH-06 | Collapse `clk_ps_axi` and `clk_mgmt` into a single domain (`pl_clk0` drives both PS-AXI master and PL register-bus). | See §4.1 |
```
**REPLACE:**
```
| FAD-ARCH-06 | Collapse `clk_ps_axi` and `clk_mgmt` into a single domain (`pl_clk0` drives both the PS-AXI master and the per-block AXI4-Lite CSR slaves). | See §4.1 |
```

### A3 — §4.4 `ps_axi`-retired sentence  ·  `arch/fad/FAD.md`

**FIND:**
```
The legacy `ps_axi` shorthand is retired per FAD-ARCH-07 — `pl_clk0` drives both the PS-AXI master and the PL register-bus, so `mgmt_if` operates entirely in `clk_mgmt`.
```
**REPLACE:**
```
The legacy `ps_axi` shorthand is retired per FAD-ARCH-07 — `pl_clk0` drives both the PS-AXI master and the per-block AXI4-Lite CSR slaves (via the AXI SmartConnect), so the entire CSR fabric operates in `clk_mgmt`.
```

### A4 — §4.4 FAD-ARCH-07 rationale paragraph  ·  `arch/fad/FAD.md`

**FIND:**
```
Per FAD-ARCH-07, `pl_clk0` drives both the PS-AXI master and the PL register-bus, collapsing what could have been two domains (`ps_axi`, `mgmt`) into one. The PS-AXI port and register-bus access have identical performance requirements (low-rate, latency-tolerant); separating them would have added a CDC boundary with no benefit. Adopted unless `mgmt_if` MS authoring identifies a specific incompatibility (OQ-§4-8).
```
**REPLACE:**
```
Per FAD-ARCH-07, `pl_clk0` drives both the PS-AXI master and the per-block AXI4-Lite CSR slaves, collapsing what could have been two domains (`ps_axi`, `mgmt`) into one. The PS-AXI port and the CSR-slave access have identical performance requirements (low-rate, latency-tolerant); separating them would have added a CDC boundary with no benefit. Adopted unless block-design / SmartConnect integration identifies a specific incompatibility (OQ-§4-8).
```

### A5 — §4.6 POR T1  ·  `arch/fad/FAD.md`

**FIND:**
```
         → mgmt_if, reg_bus, event_log, clock_ctrl operational
```
**REPLACE:**
```
         → AXI SmartConnect + per-block CSR slaves, event_log, clock_ctrl operational
```

### A6 — §4.6 POR T6  ·  `arch/fad/FAD.md`

**FIND:**
```
T = T6:  nv_cfg completes restore of CSR fields (data + module enables) via reg_bus.
```
**REPLACE:**
```
T = T6:  nv_cfg completes restore of CSR fields (data + module enables) via the AXI SmartConnect.
```

### A7 — §4.5 CDC-06 / CDC-07 owner → per-block  ·  `arch/fad/FAD.md`

**FIND:**
```
| CDC-06 | mgmt | dsp | CSR write to DSP domain | Handshake | `reg_bus` |
| CDC-07 | dsp | mgmt | CSR read response | Handshake | `reg_bus` |
```
**REPLACE:**
```
| CDC-06 | mgmt | dsp | CSR write to DSP domain | Handshake | per mixed-domain block (§4.5.1 mechanism) |
| CDC-07 | dsp | mgmt | CSR read response | Handshake | per mixed-domain block (§4.5.1 mechanism) |
```

### A8 — §5 budget negligible-consumers (POST-batch-04 anchor)  ·  `arch/fad/FAD.md`

`tle_compute` already removed by batch-04 B9; this drops `mgmt_if`, `reg_bus`.

**FIND:**
```
| Negligible consumers | `band_gain` (×2), `rf_port_sel`, `mgmt_if`, `reg_bus`, `utc_apply_engine`, `pps_capture`, `clock_ctrl` | Distributed only | < 1 BRAM each | CSRs, counters, shadow registers, small FSM state. No BRAM/URAM allocation expected. |
```
**REPLACE:**
```
| Negligible consumers | `band_gain` (×2), `rf_port_sel`, `utc_apply_engine`, `pps_capture`, `clock_ctrl` | Distributed only | < 1 BRAM each | CSRs, counters, shadow registers, small FSM state. No BRAM/URAM allocation expected. The AXI SmartConnect IP is vendor IP, budgeted separately (ADR-0012). |
```

### A9 — §6 note 6.2.i: drop `reg_bus`, keep `nv_cfg`  ·  `arch/fad/FAD.md`

**FIND:**
```
**i. Inferred infrastructure rows.**

`reg_bus` and `nv_cfg` are not literally named as internal PL blocks in SFU-001. They remain in §6 because they are architectural infrastructure required by the cited SFU functions:

- `reg_bus` is required to distribute control/status access from the management plane to PL functional blocks. `[INFERRED from SFU-001 §14, §17.1]`
- `nv_cfg` is required to implement non-volatile configuration restore. `[INFERRED from SFU-001 §17.2]`

These rows must remain marked `[INFERRED]` until an ADR or parent document names the block boundary explicitly.
```
**REPLACE:**
```
**i. Inferred infrastructure row.**

`nv_cfg` is not literally named as an internal PL block in SFU-001. It remains in §6 because it is architectural infrastructure required by a cited SFU function:

- `nv_cfg` is required to implement non-volatile configuration restore. `[INFERRED from SFU-001 §17.2]`

This row must remain marked `[INFERRED]` until an ADR or parent document names the block boundary explicitly. The former `reg_bus` block concept is retired (ADR-0012): the CSR fabric is the AMD AXI SmartConnect IP (vendor IP, not an MS) plus per-block AXI4-Lite slaves, not an SFU-owned block. The register-access *convention* itself is documented in the `register_bus` ICD.
```

### A10 — §1.2.1 functional-boundary row  ·  `arch/fad/FAD.md`

Softens "PS↔PL bridge" (which evoked the retired block); keeps "register bus" as convention.

**FIND:**
```
| Management plane (PS↔PL bridge, register bus, telemetry) | Per card | Continuous | SFU-001 §17.1 |
```
**REPLACE:**
```
| Management plane (PS↔PL CSR access, register bus, telemetry) | Per card | Continuous | SFU-001 §17.1 |
```

### A11 — §11.1 topology stub  ·  `arch/fad/FAD.md`

**FIND:**
```
*`<Block diagram: MAC → mgmt_if → register bus → per-module banks.>`*
```
**REPLACE:**
```
*`<Block diagram: PS MAC → M_AXI_HPM0_FPD (AXI4) → AXI SmartConnect → per-block AXI4-Lite CSR slaves → per-module banks.>`*
```

### A12 — §11.2 Register bus body (heading kept)  ·  `arch/fad/FAD.md`

**FIND:**
```
### 11.2 Register bus
- Protocol: `<AXI-Lite / APB>`, `<width>`-bit, see ICD.
- CDC from mgmt clock into DSP domains: per CDC inventory §4.5.
```
**REPLACE:**
```
### 11.2 Register bus
- Protocol: AXI4-Lite, 32-bit, via the AMD AXI SmartConnect fabric (ADR-0012); per-block S_AXI-Lite slaves — see [register_bus ICD](../icd/register_bus.md).
- CDC from mgmt clock into DSP domains: per-block (§4.5 CDC-06/07; mechanism §4.5.1).
```

---

## GROUP B — DDR4 reference cleanup

### B1 — §5.4 PL-DDR4 row: fix ADR-0010 misref  ·  `arch/fad/FAD.md`

ADR-0010 is the clock-tree decision; the PL-DDR4 activation trigger must not point at it.

**FIND:**
```
| **PL DDR4** (10 GB, 320b @ 333 MHz, ~76.7 Gbps usable; HW Guide §7.2) | **Reserved; activation deferred.** No MIG IP in the v0.7 bitstream; no PL DDR4 XDC; board PL DDR4 bring-up is off the critical path. | Candidate consumers: `capture` extended variant, `event_log` extended-persistence variant. Activation gated by OI-S08 resolution; triggers proposed ADR-0010 "PL DDR4 use for `capture` / `event_log` spill" at that point. | Deferred until activation. |
```
**REPLACE:**
```
| **PL DDR4** (10 GB, 320b @ 300 MHz PL clock / 2666 MT/s, ~76.7 Gbps usable; HW Guide §7.2) | **Reserved; activation deferred.** No MIG IP in the v0.7 bitstream; no PL DDR4 XDC; board PL DDR4 bring-up is off the critical path. | Candidate consumers: `capture` extended variant, `event_log` extended-persistence variant. Activation gated by OI-S08 (FAD-OQ-12); opens a dedicated PL-DDR4-use ADR at that point (number TBD — **not** ADR-0010, which is the clock-tree decision). | Deferred until activation. |
```

### B2 — §5.4 PS-DDR4 row: retire `mgmt_if`, defer the path  ·  `arch/fad/FAD.md`

**FIND:**
```
| **PS DDR4** (16 GB @ 2100 MT/s, 64+8 ECC; HW Guide §7.2) | Out of SFU PL allocation scope. | PS-owned (Linux, mgmt protocol stack, TLE/SGP4 propagation per FAD-ARCH-01). PL contact only via `mgmt_if` HPC*/HP* AXI port to push captured IQ and event records to PS for 1 GbE drain. | Capture-download bandwidth budget = `mgmt_if` MS scope. |
```
**REPLACE:**
```
| **PS DDR4** (16 GB @ 2100 MT/s, 72-bit = 64 data + 8 ECC; HW Guide §7.2) | Out of SFU PL allocation scope. | PS-owned (Linux, mgmt protocol stack, TLE/SGP4 propagation per FAD-ARCH-01). No committed SFU PL→PS bulk path: `[TBD: capture/playback bulk-data path — leaning PL-DMA → PL-DDR4 (keep high-rate IQ in PL, off the PS interconnect / 1 GbE); a PS-DDR4-via-HP path is not preferred. Resolve at capture/playback MS authoring. owner: Marco]`. | Capture-download bandwidth budget deferred with the path decision (capture/playback MS scope). |
```

### B3 — §5 closing line: ADR-0010 misref  ·  `arch/fad/FAD.md`

**FIND:**
```
`capture` and `event_log` MSs are authored under §5.3 type allocation today. The DDR-backed extended variant carries `[STUB: OI-S08]` and is gated by the proposed ADR-0010 trigger; the URAM-bounded variant is authorable now.
```
**REPLACE:**
```
`capture` and `event_log` MSs are authored under §5.3 type allocation today. The DDR-backed extended variant carries `[STUB: OI-S08]` and is gated by the PL-DDR4 activation decision (FAD-OQ-12; dedicated ADR to be assigned — not ADR-0010); the URAM-bounded variant is authorable now.
```

---

## GROUP C — OPTIONAL housekeeping change-log row  ·  `arch/fad/FAD.md`

```
| 0.7.5 | 2026-05-28 | Marco Pausini | **Retire `mgmt_if` / `reg_bus` block nouns (ADR-0012 cleanup).** The management CSR fabric is the AMD AXI SmartConnect IP + per-block AXI4-Lite slaves; the SFU-owned AXI-bridge (`mgmt_if`) and register-fabric (`reg_bus`) block concepts are retired and swept from FAD-ARCH-01/-06, §4.4, §4.6 POR, §5 budget, §11.1/§11.2, note 6.2.i, §1.2.1. CDC-06/07 owner reframed to per-block (§4.5.1). The register-bus *convention* and the `register_bus` ICD are retained. Resolves the parked R4 FAD-ARCH-01 vs FAD-ARCH-10 contradiction. DDR4 cleanup: PL-DDR4 activation trigger corrected off the erroneous ADR-0010 reference (→ FAD-OQ-12, dedicated ADR TBD); PS-DDR4 sized 16 GB/72-bit and PL-DDR4 2666 MT/s per HW Guide §7.2; the asserted PS-DDR4-via-HP capture path replaced with a deferred capture/playback-path marker (leaning PL-DMA→PL-DDR4). Flagged for ICD pass: possible `register_bus` vs `mgmt_regmap` ICD redundancy. |
```

---

## Post-apply check

```sh
# block nouns gone (ICD filename register_bus.md is the only legitimate 'reg' token)
grep -n "mgmt_if" arch/fad/FAD.md                       # expect 0
grep -n "\`reg_bus\`" arch/fad/FAD.md                   # expect 0
grep -n "AXI bridge\|register fabric\|PL register-bus" arch/fad/FAD.md   # expect 0
# kept on purpose
grep -n "register_bus ICD\|### 11.2 Register bus" arch/fad/FAD.md       # present
grep -n "per mixed-domain block (§4.5.1 mechanism)" arch/fad/FAD.md     # CDC-06/07
# DDR4
grep -n "ADR-0010" arch/fad/FAD.md   # expect: only the legitimate clock-tree references in §4.x/§5 quantities-do-not-scale, NOT the PL-DDR4 trigger lines
grep -n "not preferred. Resolve at capture/playback MS" arch/fad/FAD.md # B2 marker
grep -n "72-bit = 64 data + 8 ECC\|2666 MT/s" arch/fad/FAD.md           # sizes
```

Eyeball FAD-ARCH-01 vs FAD-ARCH-10 — they should now tell the same SmartConnect story with no `mgmt_if`/`reg_bus` block.

**Note on the `ADR-0010` grep:** ADR-0010 (clock-tree) is legitimately cited elsewhere (e.g. §4.0 "quantities that do NOT scale", §4.6 sat-switch). The check is only that the *PL-DDR4 trigger* lines no longer name it — confirm by context, not a zero count.
