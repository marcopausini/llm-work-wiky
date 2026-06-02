# beam_injector — specification

<!-- Audience: DV, firmware, integrator. Defines the contract. No internals. -->

**Status:** draft
**Traces:** `doc/reqs.md`

## Overview

`beam_injector` is a pass-through overwrite module on a 12-subband × 3-stream beam data bus. It forwards the rx interface to the tx interface unchanged, except at one selectable lane position (subband, stream, beam slot, sub-window) where it substitutes the IQ payload with samples drained from an internal injection FIFO. The FIFO is fed by an external injection source (typically a `beam_sniffer`) that has no backpressure channel; the module reports overflow and underflow as sticky status outputs. It is the dual of `beam_sniffer`: sniffer captures a window, injector writes that window back into (possibly another) stream.

## Top-level ports

<!-- managed:start regenerate-from=ports -->
### Domain: `clk_in` / `rst_in`

| Port | Dir | Width / type | Description |
|------|-----|--------------|-------------|
| `clk_in` | in | 1 | 320 MHz system clock |
| `rst_in` | in | 1 | Active-high synchronous reset |
| `injector_enable` | in | 1 | Enable. De-assert to reset state, clear sticky status, and update config. |
| `beam_xfer_rx` | rx | `fb_beam12sb_intf.rx` | 12 subbands × 3 streams, upstream |
| `beam_xfer_tx` | tx | `fb_beam12sb_intf.tx` | 12 subbands × 3 streams, downstream (pass-through with selected lane overwritten) |
| `sel_subband_in` | in | 4 | Subband select (0–11) |
| `sel_stream_in` | in | 2 | Stream select (0–2) |
| `sel_beamslot_in` | in | 4 | Beam slot select (0–15) |
| `sel_sub_beamslot_in` | in | 4 | Sub-window index within slot |
| `fft_size_in` | in | `[11:0][2:0][15:0][2:0]` | FFT size encoding per subband/stream/slot |
| `inj_sample_in` | in | `P_INPUT_DATA_WIDTH*2` (36) | Concatenated `{IM[17:0], RE[17:0]}` from injection source |
| `inj_valid_in` | in | 1 | High = push one sample into injection FIFO this cycle |
| `inj_last_in` | in | 1 | Informational end-of-window marker from source |
| `fifo_overflow_out` | out | 1 | 1-cycle pulse on each write-while-full event (passed straight through from shared FIFO). No latching — parent module accumulates in its register block if desired. |
| `fifo_underflow_out` | out | 1 | 1-cycle pulse on each window-cycle-with-empty-FIFO event. Non-latched; parent accumulates. |
<!-- managed:end -->

## Clocks and resets

| Clock | Rate | Reset | Reset style | Domain crosses |
|-------|------|-------|-------------|----------------|
| `clk_in` | 320 MHz | `rst_in` | Active-high, synchronous | None |

Single clock domain. Injection source (e.g. `beam_sniffer`) is assumed to share `clk_in`. No CDC.

## Parameters

| Parameter | Default | Valid range | Effect |
|-----------|---------|-------------|--------|
| `P_INPUT_DATA_WIDTH` | 18 | ≥1 | Width of each IQ component (RE and IM); injection sample width = 2× |
| `P_WR_PIPE_STAGES` | 2 | ≥1 | Number of pipeline register stages between rx and tx on the pass-through path |
| `P_INJECT_FIFO_DEPTH` | 256 | ≥ max window size (240) | Injection FIFO depth in samples; must absorb phase difference between source window production and injector window consumption |

## Register map

_N/A — no register interface. All configuration is via static input ports._

## Operation

**Startup**: After reset de-assertion and `injector_enable` assertion, rx→tx IQ pass-through is active immediately. The module waits for the first `eof` pulse on the selected (subband, stream) path of the rx bus before enabling any injection overwrite. This guarantees frame alignment.

**Injection source ingress**: Each cycle with `inj_valid_in = 1` pushes one sample (`inj_sample_in`) into the injection FIFO. The source has no backpressure — if the FIFO is full, the incoming sample is dropped and `fifo_overflow_out` is asserted (and stays asserted). `inj_last_in` is informational and does not gate FIFO writes.

**Window overwrite sequence**: Once frame-aligned, on each cycle where the rx bus carries `beam_valid = 1` at the (`sel_subband_in`, `sel_stream_in`) lane and `beamslot_number == sel_beamslot_in`, the module counts qualifying samples within the current slot packet. For sample indices in `[sel_sub_beamslot_in × window_size, sel_sub_beamslot_in × window_size + window_size)`, where `window_size = FFTSIZE_TO_SAMPLES[fft_size_of_selected_slot]`, the tx bus IQ at the selected lane is driven from the next FIFO read instead of the rx pass-through value. Outside the window (or outside the selected lane, slot, subband, stream), tx = rx.

**Underflow**: If a window cycle occurs while the FIFO is empty, the selected IQ position on the tx bus is driven to zero and `fifo_underflow_out` is asserted (and stays asserted). Non-IQ fields (`beam_valid`, `eof`, `beamslot_number`, `fft_size`) are always forwarded from rx to tx unchanged.

**Configuration switching**: De-assert `injector_enable` before changing any `sel_*` input or `fft_size_in`. Changing selects while enabled produces undefined behaviour.

**Enable = 0**: rx→tx IQ pass-through is forced to zero on all lanes (no samples are emitted). All internal state — FIFO contents, frame alignment, window position, sticky status outputs — is reset. The injector re-synchronises from scratch on next enable assertion. Note: this implies the injector must remain enabled during normal bus operation; disabling it squelches the entire tx bus.

## Timing diagrams

### Input frame structure

The rx `fb_beam12sb_intf` bus carries a continuous stream of 16 back-to-back packets per frame per subband. Each packet = 240 clock cycles (one beam slot's worth of samples). The injector uses this same layout to locate the overwrite window on the tx bus (structure is preserved by pass-through).

**Slot assignment across streams (per subband):**
```
Packet idx →    |   0   |   1   |   2   |  ...  |  14  |  15  |
----------------+-------+-------+-------+-------+------+------+
stream0 slot ID |   0   |   1   |   2   |  ...  |  14  |  15  |
stream1 slot ID |  16   |  17   |  18   |  ...  |  30  |  31  |
stream2 slot ID |  32   |  33   |  34   |  ...  |  46  |  47  |
```
All 3 streams advance in lockstep; each entry is the absolute slot ID carried by that stream during that packet. The `beamslot_number` signal carries the per-stream index (0–15); `sel_beamslot_in` matches against it.

**Intra-packet sample layout (240 samples) — depends on FFT mode:**

20 MHz — 1 beam, 240 contiguous samples:
```
Sample idx → |   0   |   1   | ... | 238 | 239 |
-------------+-------+-------+-----+-----+-----+
stream X     |   A   |   A   | ... |  A  |  A  |
```

10 MHz — 2 beams, 2 × 120 samples:
```
Sample idx → |   0   | ... | 119 | 120 | ... | 239 |
-------------+-------+-----+-----+-----+-----+-----+
stream X     |   A   | ... |  A  |  B  | ... |  B  |
```

5 MHz — 4 beams, 4 × 60 samples:
```
Sample idx → |  0  |...| 59 | 60 |...|119|120|...|179|180|...|239|
-------------+-----+---+----+----+---+---+---+---+---+---+---+---+
stream X     |  A  |...| A  | B  |...| B | C |...| C | D |...| D |
```

3.3 MHz — 6 beams, 6 × 40 samples:
```
Sample idx → | 0 |...|39|40|...|79|80|...|119|120|...|159|160|...|199|200|...|239|
-------------+---+---+--+--+---+--+--+---+---+---+---+---+---+---+---+---+---+---+
stream X     | A |...| A| B|...| B| C|...| C | D |...| D | E |...| E | F |...| F |
```

1.4 MHz — 12 beams, 12 × 20 samples:
```
Sample idx → |  0 |...| 19 | 20 |...| 39 | 40 |...| 59 | 60 |...| 79 | 80 |...| 99 |100 |...|119 |
-------------+----+---+----+----+---+----+----+---+----+----+---+----+----+---+----+----+---+----+
laneX        |  A |...|  A |  B |...|  B |  C |...|  C |  D |...|  D |  E |...|  E |  F |...|  F |

Sample idx → |120 |...|139 |140 |...|159 |160 |...|179 |180 |...|199 |200 |...|219 |220 |...|239 |
-------------+----+---+----+----+---+----+----+---+----+----+---+----+----+---+----+----+---+----+
laneX        |  G |...|  G |  H |...|  H |  I |...|  I |  J |...|  J |  K |...|  K |  L |...|  L |
```

**Combined frame view (pseudo-code):**
```
for packet in 0..15:            // 16 packets per frame
  for sample in 0..239:         // 240 samples per packet
    stream0 → slot = packet,         data layout = mode-dependent (above)
    stream1 → slot = 16 + packet,    data layout = mode-dependent
    stream2 → slot = 32 + packet,    data layout = mode-dependent
```
`sel_sub_beamslot_in` addresses the sub-window within the 240-sample packet: `start_sample = sel_sub_beamslot_in × window_size`, where `window_size = FFTSIZE_TO_SAMPLES[fft_size_at_selected_slot]`.

---

### Injection window (conceptual)

`sel_fft_size = 1` (10 MHz, 120-sample window). `valid_sel` = `beam_valid` on the selected lane; `slot_sel` = (`beamslot_number == sel_beamslot_in`).

```
Cycle            |  -1  |  0   |  1   |  2   | ... | 118  | 119  | 120  |
-----------------+------+------+------+------+-----+------+------+------+
clk_in (320MHz)  |  ↑   |  ↑   |  ↑   |  ↑   | ... |  ↑   |  ↑   |  ↑   |
valid_sel        |  1   |  1   |  1   |  1   | ... |  1   |  1   |  1   |
slot_sel         |  1   |  1   |  1   |  1   | ... |  1   |  1   |  1   |
sample_idx       |  N-1 |  N   | N+1  | N+2  | ... |N+118 |N+119 |N+120 |
rx IQ (sel lane) | R_-1 | R_0  | R_1  | R_2  | ... |R_118 |R_119 |R_120 |
FIFO out (read)  |  --  | F_0  | F_1  | F_2  | ... |F_118 |F_119 |  --  |
tx IQ (sel lane) | R_-1 | F_0  | F_1  | F_2  | ... |F_118 |F_119 |R_120 |
```
where `N = sel_sub_beamslot_in × window_size`. Between qualifying cycles (other subbands/streams/slots, or `beam_valid = 0`) the FIFO is not advanced and tx IQ at the selected lane equals rx IQ.

### Underflow

If at any cycle in the range `[0, 119]` the FIFO is empty, `tx IQ (sel lane)` that cycle is 0 and `fifo_underflow_out` latches to 1. The FIFO is not advanced on empty. Non-underflow cycles of the same window still draw the next available FIFO sample — underflow zero-fills only the empty cycles.

## Error and status

- **Status bits:**
  - `fifo_overflow_out` — 1-cycle pulse on each write-while-full on the injection FIFO. Not latched inside this block.
  - `fifo_underflow_out` — 1-cycle pulse on each window cycle where the FIFO is empty. Not latched.
  - Both pulses are driven directly from the shared-FIFO primitive; the parent IP (instantiator) is expected to aggregate them into a register-block counter or sticky bit.
- **Interrupts:** _N/A — no interrupt interface._
- **Illegal-access behaviour:**
  - Out-of-range `sel_subband_in` / `sel_stream_in`: undefined. The module does not bounds-check selects against bus dimensions.
  - Changing `sel_*` while `injector_enable = 1`: undefined. Caller must de-assert enable first.

## Performance

| Metric | Value | Condition |
|--------|-------|-----------|
| Pass-through throughput | 1 frame / frame, full bus | Always (independent of injector state) |
| Pass-through latency (rx → tx) | `P_WR_PIPE_STAGES + 1` cycles | Fixed for all lanes |
| Injection throughput | 1 sample / qualifying cycle | During active window |
| Injection window samples | `FFTSIZE_TO_SAMPLES[fft_size]` | 20 / 40 / 60 / 120 / 240 |
| Injection FIFO capacity | `P_INJECT_FIFO_DEPTH` | ≥ largest window size (240) |

## Test plan hooks

- **R1:** Drive arbitrary rx bus; verify tx equals rx on all non-IQ fields and on all non-selected lanes.
- **R2:** Drive stimuli on all 12×3 paths; verify only the selected (subband, stream, slot, sub-slot) IQ position is modified.
- **R3:** Sweep `sel_subband_in` / `sel_stream_in` / `sel_beamslot_in` / `sel_sub_beamslot_in`; verify correct lane is targeted.
- **R4:** Pre-fill FIFO; verify window starts exactly at `sel_sub_beamslot_in × window_size` on the qualifying packet.
- **R5:** Verify `FFTSIZE_TO_SAMPLES` lookup for all five supported encodings.
- **R6:** Push injection stream; verify each `inj_valid_in = 1` enqueues one sample and the FIFO drains in order.
- **R7:** Verify exactly `window_size` samples are consumed from the FIFO per injection window and that the tx IQ at the selected lane matches the consumed sequence.
- **R8:** Starve the FIFO during a window; verify tx IQ at the selected lane is zero on the empty cycles and that `fifo_underflow_out` latches high.
- **R9:** Write to the FIFO while full; verify the incoming sample is dropped and `fifo_overflow_out` latches high.
- **R10:** Toggle `injector_enable`; verify FIFO clears, sticky status clears, and the injector re-waits for `eof` before re-injecting.
- **R11:** Assert `injector_enable` mid-frame; verify no injection occurs before the first subsequent `eof`, but rx→tx IQ pass-through resumes immediately after enable.

## Open questions

- Confirm `P_INJECT_FIFO_DEPTH` default of 256 is adequate for worst-case producer↔consumer phase offset.
- `beam_sniffer.last_beam_out` is tied to the sniffer's selected async window — it is **not** aligned to `eof` or to the start of the 240-cycle packet on the bus. If the injector consumes samples from a sniffer, the injected window may land with a phase offset relative to the injector's tx `eof` / packet boundary. Open: does that phase offset break downstream consumers? If yes, the injection-source contract needs a frame-aligned marker (e.g. replace sniffer's `last` with an `eof`-tick, or add a separate sync line).

## References

- `doc/reqs.md`
- `rtl/src/filter_bank_shared/fb_beams_intf.sv` — `fb_beam12sb_intf` (shared `.rx` / `.tx` modports)
- `rtl/src/filter_bank_shared/filter_bank_pkg.sv` — `FFTSIZE_TO_SAMPLES`, `P_BEAMSLOTS_PER_SB`
- `rtl/src/beam_sniffer/doc/spec.md` — frame and slot layout (reference; structure reused)
 