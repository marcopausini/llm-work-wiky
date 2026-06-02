# Change-set — FAD batch 06 (Chapter 4 review: clocking / reset / CDC / RDC)

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Scope:** FAD §4 review. Anchors are byte-exact against current §4 text (which batches 04/05 did **not** touch, except the two ID hunks below which use batch-independent minimal anchors).
**Apply method:** `str_replace` exact find-replace. Preserve `→ × ± § ↔ ²` glyphs, backticks, and the non-breaking spaces in `1 024` / `1 048.576`.
**Pre-flight:** order-independent w.r.t. batches 04/05.

Reviewer items already closed (no hunks): Aurora 19.2→19.6608 (batch 02), CDC-06/07 `reg_bus` owner (batch 05 A7 → per-block), 1PPS coherency (batch 04 F1), `ADR-0002`/`OQ-§4-3` assumption (does not exist in current FAD — stale).

Groups: **A** MMCM notation · **B** clk_rfdc sync clarification · **C** Aurora retiming · **D** reset source + RDC table · **E** startup scope · **F** stale-ID cleanup.

---

## GROUP A — MMCM notation (high; deterministic)

`M=4, D=3` is the 4/3 ratio, not Xilinx MMCM fields. Correct primitive config: `DIVCLK_DIVIDE=1, CLKFBOUT_MULT=4, CLKOUT0_DIVIDE=3`; VCO = 4 × clk_rfdc_axis = 1024 / 1048.576 MHz; clk_dsp = VCO/3 = 341.333 / 349.525 MHz.

### A1 — §4.0 gearbox sentence  ·  `arch/fad/FAD.md`

**FIND:**
```
`clk_dsp = AXIS × 4 / 3` via a single MMCM with `M=4, D=3` — a synchronous 4↔3 sample gearbox between the RFdc-tile boundary (4 samples/clk) and the DSP fabric (3 samples/clk), with no async FIFO.
```
**REPLACE:**
```
`clk_dsp = clk_rfdc_axis × 4/3` via a single MMCM (`DIVCLK_DIVIDE=1, CLKFBOUT_MULT=4, CLKOUT0_DIVIDE=3`; VCO = 4 × clk_rfdc_axis) — a synchronous 4↔3 sample gearbox between the RFdc-tile boundary (4 samples/clk) and the DSP fabric (3 samples/clk), with no async FIFO.
```

### A2 — §4.2.1 clk_dsp clock-table row  ·  `arch/fad/FAD.md`

**FIND:**
```
| `clk_dsp` | 341.333 MHz | 349.525 MHz | PL MMCM, fixed `M=4, D=3` from `clk_rfdc_axis` | Entire DSP datapath (every block in §6 with `dsp` shorthand) | yes |
```
**REPLACE:**
```
| `clk_dsp` | 341.333 MHz | 349.525 MHz | PL MMCM (`DIVCLK_DIVIDE=1, CLKFBOUT_MULT=4, CLKOUT0_DIVIDE=3`) from `clk_rfdc_axis` | Entire DSP datapath (every block in §6 with `dsp` shorthand) | yes |
```

### A3 — §4.2.1 MMCM cascade step 7  ·  `arch/fad/FAD.md`

**FIND:**
```
7. **PL MMCM (`M=4, D=3`, VCO = 1 024 / 1 048.576 MHz) → BUFG → `clk_dsp`.** Single MMCM cascade per UG949. The MMCM VCO frequency must be valid at both sample-rate points.
```
**REPLACE:**
```
7. **PL MMCM → BUFG → `clk_dsp`.** Single MMCM cascade per UG949: `DIVCLK_DIVIDE=1, CLKFBOUT_MULT=4, CLKOUT0_DIVIDE=3` → VCO = 4 × clk_rfdc_axis = 1 024 MHz (ASIC) / 1 048.576 MHz (FPGA); `clk_dsp` = VCO/3 = 341.333 / 349.525 MHz. Fixed fields → `clk_dsp` tracks the ×1.024 input scaling automatically (FAD-ARCH-08). VCO is within the MMCM Fvco range at both sample-rate points.
```

---

## GROUP B — clk_rfdc_axis ↔ clk_dsp synchronous (high; clarify, keep final)

### B1 — §4.2.1 no-async-FIFO sentence  ·  `arch/fad/FAD.md`

**FIND:**
```
`clk_rfdc_axis ↔ clk_dsp` is synchronous-derived through the fixed-ratio MMCM (Vivado infers it from `create_generated_clock`), so no async FIFO is needed at the `rfdc_wrap` boundary.
```
**REPLACE:**
```
`clk_rfdc_axis ↔ clk_dsp` is synchronous-derived through the fixed-ratio MMCM (Vivado infers it from `create_generated_clock`), so no async FIFO is needed at the `rfdc_wrap` boundary; the 4↔3 sample-count mismatch is absorbed by a synchronous elastic/gearbox register stage inside `rfdc_wrap` (uArch), not a clock-domain FIFO.
```

---

## GROUP C — Aurora multi-lane retiming (high; resolve OR)

### C1 — §4.2.2 retiming sentence  ·  `arch/fad/FAD.md`

Resolves the false "FIFO **or** obg_phase_align" OR: clock crossing is per-lane FIFO in `obg_link`; inter-lane deskew is `obg_phase_align` (downstream, in `clk_dsp`).

**FIND:**
```
One Aurora user_clk acts as timing master for the four-lane group; the other three lane user_clks are re-timed via per-lane FIFOs or via `obg_phase_align` (see `obg_link` MS).
```
**REPLACE:**
```
The four OBG lanes have independent RX recovered clocks (the matrix may source them from different DCUs). One lane is selected at commissioning as the SFU timing-master reference — its CDR drives the Si5394 loop (INV-14) — and that selection is independent of data retiming. For the data path, each lane crosses from its `aurora_user` clock into `clk_dsp` through a per-lane elastic/async FIFO in `obg_link` (CDC-01/-02). Inter-lane sample skew is then removed downstream in `clk_dsp` by `obg_phase_align` (per-lane programmable delay, INV-05; SFU-001 §6.2). Clock crossing (`obg_link`) and deskew (`obg_phase_align`) are distinct, sequential functions — not an either/or.
```

---

## GROUP D — reset source correction + RDC inventory (blocker)

### D1 — §4.4.2 `rst_dsp_ctrl_n` source  ·  `arch/fad/FAD.md`

Corrects the "generated in `clk_mgmt`" attribution: the source is PS `pl_resetn0` via the per-domain `proc_sys_reset`, release gated on DSP MMCM lock; `clock_ctrl` only asserts during sat-switch.

**FIND:**
```
| `rst_dsp_ctrl_n` | `clock_ctrl`, gated on DSP MMCM lock and sat-switch state. Generated in `clk_mgmt`, crossed into `clk_dsp` via reset bridge. | `clk_dsp` | Control logic of all `clk_dsp` blocks. Does NOT reset DSP datapath pipeline registers (R4). |
```
**REPLACE:**
```
| `rst_dsp_ctrl_n` | PS `pl_resetn0` via the `clk_dsp` `proc_sys_reset`; release gated on DSP MMCM lock. Additionally asserted by `clock_ctrl` during sat-switch. Enters `clk_dsp` through the reset bridge (R2). | `clk_dsp` | Control logic of all `clk_dsp` blocks. Does NOT reset DSP datapath pipeline registers (R4). |
```

### D2 — new §4.5.4 Reset-domain-crossing (RDC) inventory  ·  `arch/fad/FAD.md`

Adds the RDC table the CDC sign-off requires (reset crossings have distinct async-assert / sync-deassert analysis). Inserted at the end of the CDC-inventory subsection.

**FIND:**
```
**Rows added when their MSs are authored:** `nv_cfg` QSPI ↔ `clk_mgmt` (encapsulated in AMD QSPI IP); `clock_ctrl` secondary-mode reference path (likely encapsulated in MMCM/PLL config).

---
```
**REPLACE:**
```
**Rows added when their MSs are authored:** `nv_cfg` QSPI ↔ `clk_mgmt` (encapsulated in AMD QSPI IP); `clock_ctrl` secondary-mode reference path (likely encapsulated in MMCM/PLL config).

#### 4.5.4 Reset-domain crossings (RDC)

Reset crossings are analysed separately from data/control CDC: every reset entering a domain from an asynchronous source or another clock domain uses **asynchronous assertion / synchronous deassertion** through a reset bridge (R2, §4.4.3), so the analysis is edge-integrity and release-timing, not data coherence.

| # | Reset | Source → target domain | Mechanism | Owner |
|---|---|---|---|---|
| RDC-01 | `rst_mgmt_n` | PS `pl_resetn0` (async) → `clk_mgmt` | `proc_sys_reset` (async-assert / 2-flop sync-deassert, R2) | `proc_sys_reset` (`clk_mgmt` instance) |
| RDC-02 | `rst_dsp_ctrl_n` (POR) | PS `pl_resetn0` (async) → `clk_dsp`, release gated on DSP MMCM `dcm_locked` | `proc_sys_reset` (async-assert / 2-flop sync-deassert, R2) | `proc_sys_reset` (`clk_dsp` instance) |
| RDC-03 | `rst_dsp_ctrl_n` (sat-switch) | `clock_ctrl` request (`clk_mgmt`) → `clk_dsp` | Request synchronised mgmt→dsp (2-flop); drives reset bridge (async-assert / sync-deassert) | `clock_ctrl` + `proc_sys_reset` (`clk_dsp`) |
| RDC-04 | `rst_rfdc_n` | `clock_ctrl` trigger (`clk_mgmt`) → RFdc IP reset FSM (`clk_rfdc_axis`) | Trigger synchronised mgmt→rfdc_axis (2-flop); IP-managed reset FSM (R6) | `rfdc_wrap` (IP) + `clock_ctrl` |
| RDC-05 | `rst_aurora_n` | Aurora IP reset FSM (`clk_aurora_user`) | IP-managed (R6); no SFU cross-domain assertion in operation | `obg_link` (IP) |

Per-block CSR-reset behaviour (R5: CSR data fields not reset) and the no-`rst_dsp_data_n` policy (R4) are stated in §4.4.2 and are not RDC rows. Detailed `proc_sys_reset` instancing and the `clock_ctrl` sat-switch assertion sequence are `clock_ctrl` MS scope; this inventory fixes the crossings and owners.

---
```

---

## GROUP E — startup reference scope (high; out of FAD PL scope)

### E1 — §4.2.1 startup disclaimer  ·  `arch/fad/FAD.md`

Adds the scope boundary immediately after the clock cascade (the B1 sentence). **Apply B1 first** — this FIND targets the B1 output.

**FIND:**
```
`clk_rfdc_axis ↔ clk_dsp` is synchronous-derived through the fixed-ratio MMCM (Vivado infers it from `create_generated_clock`), so no async FIFO is needed at the `rfdc_wrap` boundary; the 4↔3 sample-count mismatch is absorbed by a synchronous elastic/gearbox register stage inside `rfdc_wrap` (uArch), not a clock-domain FIFO.
```
**REPLACE:**
```
`clk_rfdc_axis ↔ clk_dsp` is synchronous-derived through the fixed-ratio MMCM (Vivado infers it from `create_generated_clock`), so no async FIFO is needed at the `rfdc_wrap` boundary; the 4↔3 sample-count mismatch is absorbed by a synchronous elastic/gearbox register stage inside `rfdc_wrap` (uArch), not a clock-domain FIFO.

**Startup reference — out of FAD PL scope.** Before the recovered-clock loop is closed, MGT REFCLK availability (Si5394 free-run / holdover / board default) and the GT-CDR lock transition are board-design and PS-clock-bring-up concerns, owned by the board + PS clock-setup software + the `clock_ctrl` MS (ADR-0010). The FAD PL architecture assumes a locked reference at the GT input; the bootstrap state machine is not FAD scope. `[INFERRED: Si5394 free-run/holdover startup per HW Guide §8.2.x; expiry: clock_ctrl MS design_ready.]`
```

---

## GROUP F — stale-ID cleanup (medium)

### F1 — §4.4 clk-collapse ID (×1, order-independent anchor)  ·  `arch/fad/FAD.md`

clk_ps_axi/clk_mgmt collapse is FAD-ARCH-06, not -07 (07 = filter-bank). Anchor stable across batch 05.

**FIND:**
```
retired per FAD-ARCH-07 —
```
**REPLACE:**
```
retired per FAD-ARCH-06 —
```

### F2 — §4.4 clk-collapse ID (×1, order-independent anchor)  ·  `arch/fad/FAD.md`

**FIND:**
```
Per FAD-ARCH-07, `pl_clk0`
```
**REPLACE:**
```
Per FAD-ARCH-06, `pl_clk0`
```

### F3 — §4.2.1 drop legacy FAD-DEC-18  ·  `arch/fad/FAD.md`

Aurora user-clock/duty is a derived fact (Aurora wizard / line rate), not a decision.

**FIND:**
```
user_width = 64, so `RXOUTCLK = TXOUTCLK = user_clk = 19 660.8 / 64 = 307.2 MHz` with 32/33 valid duty (FAD-DEC-18).
```
**REPLACE:**
```
user_width = 64, so `RXOUTCLK = TXOUTCLK = user_clk = 19 660.8 / 64 = 307.2 MHz` with 32/33 valid duty (Aurora 64B/66B wizard default at this line rate).
```

### F4 — §4.2.4 drop legacy FAD-DEC-20 (transport paths intro)  ·  `arch/fad/FAD.md`

Captured in the §4.6 `clock_ctrl` narrative + ADR-0010; no decision ID.

**FIND:**
```
PS-side software writes every clock device. Two transport paths exist, with different failure modes (FAD-DEC-20):
```
**REPLACE:**
```
PS-side software writes every clock device. Two transport paths exist, with different failure modes (see §4.6 `clock_ctrl` and ADR-0010):
```

### F5 — §4.4.4 drop legacy FAD-DEC-20 (POR sequence)  ·  `arch/fad/FAD.md`

**FIND:**
```
         + LMK04821 + LMX2594 ×2 + RFdc tile PLL (via bittware rfx_clock_setup.elf over PS → PL SPI bridge, per FAD-DEC-20)
```
**REPLACE:**
```
         + LMK04821 + LMX2594 ×2 + RFdc tile PLL (via bittware rfx_clock_setup.elf over PS → PL SPI bridge)
```

### F6 — §4.4 reset-policy ID (FAD-DEC-16 → §4.4 rules)  ·  `arch/fad/FAD.md`

`DEC-16` maps to ARCH-10 (CSR fabric) in §15 — wrong target for the reset policy. The control-path-only reset policy now *is* §4.4 rules R1–R7.

**FIND:**
```
This minimises reset fanout, allows SRL/DSP/BRAM packing, and matches UG949 best practice(FAD-DEC-16).
```
**REPLACE:**
```
This minimises reset fanout, allows SRL/DSP/BRAM packing, and matches UG949 best practice (§4.4 rules R1–R7).
```

---

## GROUP G — OPTIONAL change-log row  ·  `arch/fad/FAD.md`

```
| 0.7.6 | 2026-05-28 | Marco Pausini | **Chapter-4 review fixes (pre-G1).** MMCM notation corrected from the ambiguous `M=4, D=3` to explicit `DIVCLK_DIVIDE=1, CLKFBOUT_MULT=4, CLKOUT0_DIVIDE=3` (VCO = 4×clk_rfdc_axis = 1024/1048.576 MHz; clk_dsp = VCO/3) in §4.0/§4.2.1. `clk_rfdc_axis↔clk_dsp` synchronous claim kept final with the 4↔3 gearbox clarified as a synchronous elastic stage in `rfdc_wrap` (uArch), not an async FIFO. Aurora multi-lane retiming de-OR'd: per-lane clock crossing in `obg_link` (CDC-01/-02) vs inter-lane deskew in `obg_phase_align` (INV-05), sequential not either/or. Reset: `rst_dsp_ctrl_n` source corrected to PS `pl_resetn0` via `clk_dsp` `proc_sys_reset` (gated on DSP MMCM lock; `clock_ctrl` asserts on sat-switch); new §4.5.4 RDC inventory (RDC-01..05) added. Startup MGT-REFCLK reference scoped out of FAD PL (board / PS clock bring-up / `clock_ctrl` MS). Stale IDs cleaned: FAD-ARCH-07→06 (clk collapse) ×2; legacy FAD-DEC-16/18/20 tags dropped (folded to §4.4 R1–R7 / Aurora-wizard fact / §4.6+ADR-0010). Reviewer items already closed elsewhere: Aurora 19.6608 (batch 02), CDC-06/07 owner (batch 05), 1PPS coherency (batch 04); ADR-0002/OQ-§4-3 marker stale (absent from current FAD). |
```

---

## Post-apply check

```sh
grep -c "M=4, D=3" arch/fad/FAD.md                              # expect 0
grep -n "CLKFBOUT_MULT=4, CLKOUT0_DIVIDE=3" arch/fad/FAD.md     # A1/A2/A3
grep -n "synchronous elastic/gearbox register stage" arch/fad/FAD.md   # B1
grep -n "not an either/or" arch/fad/FAD.md                      # C1
grep -n "via the \`clk_dsp\` \`proc_sys_reset\`" arch/fad/FAD.md # D1
grep -n "#### 4.5.4 Reset-domain crossings" arch/fad/FAD.md     # D2
grep -c "RDC-0[1-5]" arch/fad/FAD.md                            # expect 5
grep -n "Startup reference — out of FAD PL scope" arch/fad/FAD.md  # E1
grep -c "FAD-ARCH-07" arch/fad/FAD.md                           # expect 1 (only the def row §1.4.1)
grep -c "FAD-DEC-1\|FAD-DEC-2" arch/fad/FAD.md                  # expect 0 in §4 body
```

`FAD-ARCH-07` should remain exactly once — its definition row in §1.4.1 (filter-bank). Confirm by context.
