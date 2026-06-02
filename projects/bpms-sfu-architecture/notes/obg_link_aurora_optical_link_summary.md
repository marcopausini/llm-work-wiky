# Optical Link, Aurora, SERDES, OBG Link, and Clock Recovery — High-Level Summary

**Context:** BPMS 1.0 SFU FPGA architecture  
**Purpose:** Build a simple mental model for the optical/Aurora/OBG interface before deriving the `obg_link` Module Spec and FAD clocking section.

---

## 1. Starting point: what is transferred from DCU to SFU?

In the BPMS downlink path, the **DCU** sends **spectral bins** to the **SFU**.

The SFU does not receive LTE time-domain waveforms directly from the DCU. It receives frequency-domain bin data already prepared by the DCU. The SFU then selects bins, routes them to the correct sub-band, reconstructs time-domain wideband signals through the filter bank, applies band-level corrections, and drives the RF output.

A simplified view is:

```text
DCU
  -> OBG spectral bins
  -> Aurora over SERDES
  -> optical fiber / optical matrix
  -> SFU
  -> bins selection
  -> filter bank synthesis
  -> band gain / band doppler
  -> RF output
```

The reverse happens on the uplink: the SFU digitizes RF input, decomposes it into bins, and sends those bins back to the DCU.

---

## 2. What is an optical link?

An **optical link** is the physical communication path that carries digital data as light through fiber.

In this system, it includes:

```text
FPGA SERDES electrical interface
  -> optical transceiver / QSFP28 module
  -> fiber cable
  -> optical matrix / switch
  -> fiber cable
  -> optical transceiver / QSFP28 module
  -> FPGA SERDES electrical interface
```

So yes, in this context, the optical link is essentially a **fiber-optic connection** between the DCU side and the SFU side, routed through the optical matrix.

The optical matrix should be thought of as a **reconfigurable fiber patch panel**. It routes optical paths, but it does not decode Aurora, understand OBG frames, inspect bins, or perform DSP.

Important distinction:

```text
Optical link = physical transport medium
Aurora       = digital link protocol running over that physical transport
OBG          = BPMS-specific payload/framing concept carried by Aurora
```

---

## 3. What is SERDES?

**SERDES** means **serializer/deserializer**.

Inside the FPGA, logic normally works with parallel words:

```text
64-bit, 128-bit, 256-bit, etc.
```

But a high-speed optical lane carries data serially:

```text
bit, bit, bit, bit, bit, ...
```

The SERDES converts between these two forms.

### Transmit direction

```text
parallel FPGA data
  -> serializer
  -> high-speed serial bitstream
  -> optical module
  -> light over fiber
```

### Receive direction

```text
light over fiber
  -> optical module
  -> high-speed serial bitstream
  -> deserializer
  -> parallel FPGA data
```

For the SFU, the OBG interface uses multiple high-speed serial lanes. The architecture describes **4 independent Aurora SERDES lanes** on the Q0 optical interface.

---

## 4. What is Aurora?

**Aurora** is an AMD/Xilinx FPGA link-layer protocol used to move data across high-speed serial transceiver lanes.

Aurora is not the fiber. It is not the optical module. It is the protocol implemented in FPGA logic and GT transceivers to make the serial link usable.

A useful analogy:

```text
Fiber   = road
SERDES  = vehicle engine / transmission
Aurora  = traffic rules and framing
OBG     = cargo being transported
```

Aurora provides the link behavior needed to transport data reliably between FPGAs: lane operation, framing, alignment, link status, and a user-side data interface into the FPGA fabric.

For BPMS, Aurora carries the **OBG frame data**, and the OBG frame data contains the **spectral bins**.

---

## 5. What is an Aurora lane?

A **lane** is one high-speed serial channel.

For the SFU Q0 interface:

```text
Q0 QSFP28 port = 4 high-speed optical lanes
```

Each lane is independent at the serial/Aurora level.

Conceptually:

```text
Lane 0 -> OBG frame stream
Lane 1 -> OBG frame stream
Lane 2 -> OBG frame stream
Lane 3 -> OBG frame stream
```

Each lane carries a portion of the total OBG transport capacity. In the SFU architecture, each OBG lane carries up to **5,760 spectral bins per frame**.

So at a high level:

```text
4 lanes x 5,760 bins/frame = 23,040 bins/frame total transport capacity
```

Do not confuse this with the filter-bank sub-band bin count. The OBG transport lanes and the internal filter-bank bin organization are related by the bins selector, but they are not the same architectural object.

---

## 6. What is OBG?

**OBG** means **Optical Beam Group**.

For this architecture, the practical meaning is:

```text
OBG = BPMS-defined grouping and framing of spectral bins transported between DCU and SFU
```

The OBG concept is above Aurora. Aurora is the generic transport protocol. OBG defines what BPMS is transporting.

So:

```text
Aurora = how the bits are moved across the link
OBG    = what those bits mean in the BPMS architecture
```

The OBG payload contains spectral bins. Those bins may belong to AxCs of different LTE bandwidths, because the DCU packs bins at bin granularity.

---

## 7. Is `obg_link` the correct module name?

Yes. **`obg_link` is the recommended module name** for the FAD and Module Spec inventory.

Reason: the block is not just an Aurora wrapper. It is the SFU-side termination of the complete BPMS OBG transport path:

```text
Q0 optical lanes
  -> SERDES
  -> Aurora protocol
  -> OBG frame handling
  -> CDC into DSP domain
  -> internal bin stream
```

Alternative names were considered:

| Name | Assessment | Reason |
|---|---|---|
| `obg_link` | Recommended | Captures the BPMS function and is concise |
| `obg_aurora_link` | Acceptable | More explicit, but longer |
| `aurora_link` | Too generic | Describes the protocol, not the BPMS payload/function |
| `obg_phy` | Too low-level | Sounds like only the physical/SERDES layer |
| `obg_if` | Too vague | Could mean protocol, registers, or a top-level interface |

Recommendation:

```text
Keep `obg_link`.
```

Use the Module Spec text to make clear that it includes Aurora/SERDES termination, OBG frame transport, CDC, clock recovery status, and management/status reporting.

---

## 8. What does `obg_link` do?

At high level, `obg_link` is the boundary between the external optical/Aurora world and the internal SFU DSP world.

It should perform these operations:

### External link termination

```text
1. Terminate 4 full-duplex Aurora SERDES lanes.
2. Receive DL OBG frames from the DCU.
3. Transmit UL OBG frames back to the DCU.
4. Monitor Aurora link status and lane health.
```

### Clocking and timing

```text
5. Recover timing from a configured DL timing-master lane using CDR.
6. Report CDR lock and selected timing-master lane status.
7. Provide timing information to the clock-control architecture.
```

### Framing and payload handling

```text
8. Preserve lane identity.
9. Preserve frame boundaries.
10. Present received DL OBG payloads as internal bin streams.
11. Accept UL bin streams and frame them for Aurora transmission.
12. Detect and report framing/protocol errors.
```

### CDC

```text
13. Cross DL data from the Aurora RX/user clock domain into the DSP clock domain.
14. Cross UL data from the DSP clock domain into the Aurora TX/user clock domain.
15. Report overflow, underflow, and CDC-related status.
```

At MS level, the CDC **mechanism** should be specified. FIFO sizing and internal implementation details remain uArch scope.

---

## 9. High-level `obg_link` input and output interfaces

The block has four main interface groups.

---

### 9.1 External high-speed serial interface

These are not ordinary RTL data buses. They are GT/transceiver serial ports connected to the Q0 optical interface.

Conceptual ports:

```systemverilog
input  logic [3:0] obg_rx_p;
input  logic [3:0] obg_rx_n;
output logic [3:0] obg_tx_p;
output logic [3:0] obg_tx_n;
```

Meaning:

```text
DL: DCU -> optical matrix -> SFU RX lanes
UL: SFU TX lanes -> optical matrix -> DCU
```

The link is full-duplex, so DL and UL traffic exist simultaneously over the same physical lane group, using opposite directions.

---

### 9.2 Internal DL output toward DSP

This is the user-side stream from `obg_link` into the SFU downlink processing chain.

Flow:

```text
obg_link -> obg_phase_align -> bins_sel_dl
```

Conceptual stream:

```text
obg_dl_stream_o
```

Payload should include, or allow reconstruction of:

```text
lane_id
frame boundary / eof / last
bin payload: complex IQ spectral bin data
valid / ready handshake
error sideband, if required
```

The exact stream format belongs in the `obg_frame` ICD and/or `streaming_bus` ICD.

---

### 9.3 Internal UL input from DSP

This is the user-side stream from the SFU uplink bins selector back into the OBG/Aurora transmit path.

Flow:

```text
bins_sel_ul -> obg_link -> Aurora TX -> optical link -> DCU
```

Conceptual stream:

```text
obg_ul_stream_i
```

Payload should include:

```text
lane_id or destination lane mapping
frame boundary / eof / last
complex bin payload
valid / ready handshake
error/status sideband, if required
```

---

### 9.4 Management and status interface

`obg_link` must expose control and telemetry because it owns link status and participates in timing recovery.

Minimum conceptual controls:

```text
enable / reset controls
timing_master_lane_select
loopback/debug mode
alarm masks
status clear / counter clear
```

Minimum conceptual status:

```text
cdr_lock[3:0]
aurora_link_up[3:0]
lane_error counters
frame_error counters
rx_overflow
tx_underflow
selected timing-master lane
active clock-source status
```

The exact register fields are designer/register-map scope later, but the Module Spec must define the minimum externally visible behavior.

---

## 10. What is clock recovery from Aurora?

On a high-speed serial link, the receiver usually does not receive a separate clock wire. It receives only the serial bitstream.

The transceiver performs **CDR — Clock and Data Recovery**.

Conceptually:

```text
incoming serial bitstream
  -> CDR locks to bit transitions
  -> recovered serial timing
  -> deserializer creates parallel words
  -> Aurora creates user-side data stream
```

For the SFU primary clock mode, one configured OBG DL lane acts as the timing-master lane. The SFU recovers timing from that lane and derives internal clocks from it.

Important distinction:

```text
Recovered serial timing is not necessarily the DSP clock directly.
```

The recovered timing goes through the GT/Aurora clocking architecture and potentially PLL/MMCM clock generation before becoming the clocks used by SFU logic and converters.

---

## 11. How do we know the recovered clock frequency?

At high level, it is derived from:

```text
Aurora serial line rate
Aurora encoding mode
GT datapath width
Aurora user-clock configuration
clock dividers / PLL / MMCM settings
```

The architecture gives the line-rate-level fact:

```text
Q0 OBG interface = 4 Aurora lanes at approximately 19.2 Gbps per lane
```

But the final FPGA user clocks are not known just from that statement.

For example, the recovered serial clock is tied to the 19.2 Gbps serial stream, but the Aurora user clock might be a divided parallel clock depending on Aurora IP configuration.

Therefore, the correct spec posture is:

```text
Do not invent the Aurora user-clock frequency.
```

In FAD §4 and the `obg_link` MS, use TBDs until the Aurora IP configuration is fixed:

```text
aurora_rx_clk:
  Frequency: [TBD: derived from Aurora 64B/66B IP configuration for 19.2 Gbps lane rate]
  Source: GT/Aurora recovered clocking

aurora_tx_clk:
  Frequency: [TBD: derived from Aurora 64B/66B IP configuration]
  Source: GT/Aurora transmit clocking

dsp_clk:
  Frequency: [TBD: SFU DSP clock plan]
  Source in primary mode: derived from selected OBG DL lane CDR
  Source in secondary mode: derived from external 10 MHz CI reference
```

The source documents define the architecture and source of timing. They do not fully define all internal generated clock frequencies.

---

## 12. What clock should `obg_link` run on?

`obg_link` does not run on one clock. It spans multiple clock domains.

Minimum clock domains:

| Clock domain | Purpose | Source |
|---|---|---|
| `gt_refclk` | GT transceiver reference | Board / transceiver clocking |
| `aurora_rx_clk` | Aurora RX user-side logic | Recovered from incoming DL lane timing |
| `aurora_tx_clk` | Aurora TX user-side logic | Aurora/GT transmit clocking |
| `dsp_clk` | Internal SFU stream interface | SFU DSP/system clock |
| `mgmt_clk` | Register/status access | PS/PL management clock |

Primary clock mode:

```text
selected OBG DL timing-master lane
  -> CDR
  -> recovered timing
  -> clock generation
  -> SFU FPGA logic and ADC/DAC clocking
```

Secondary clock mode:

```text
external 10 MHz CI reference
  -> clock generation
  -> SFU FPGA logic and ADC/DAC clocking
```

For the Module Spec, the important contract is:

```text
`obg_link` shall expose an Aurora-side clock domain and a DSP-side clock domain.
DL data shall cross from Aurora RX/user clock domain to DSP clock domain through an approved CDC mechanism.
UL data shall cross from DSP clock domain to Aurora TX/user clock domain through an approved CDC mechanism.
```

---

## 13. Recommended MS-level description of `obg_link`

A clean Module Spec description would be:

```text
`obg_link` terminates the four full-duplex OBG Aurora lanes on Q0. It receives DL OBG frames from the DCU, recovers the primary timing reference from a configured DL timing-master lane, crosses received OBG frame payloads into the SFU DSP clock domain, and emits DSP-side DL OBG bin streams toward `obg_phase_align`.

In the UL direction, it accepts DSP-side OBG bin/frame streams from `bins_sel_ul`, crosses them into the Aurora transmit domain, frames them for OBG/Aurora transport, and transmits them back to the DCU over the same four physical lanes.

The block reports CDR lock, Aurora link status, frame/protocol errors, overflow/underflow events, selected timing-master lane, and active clock-source status through the management interface.
```

---

## 14. What to specify now vs later

### Specify now in FAD / MS

```text
Block name: `obg_link`
Functional role
DL and UL direction ownership
External lane count: 4 full-duplex Aurora lanes
OBG frame/bin payload responsibility
Clock-domain boundary: Aurora side <-> DSP side
Approved CDC mechanism class
Required management/status observability
Relationship to primary clock recovery
```

### Leave TBD until Aurora IP configuration is fixed

```text
Exact Aurora user-clock frequency
Exact GT reference-clock requirements
Exact internal Aurora datapath width
Exact FIFO depths
Exact register bit layout
Exact lane error counter widths
Exact framing signal layout, pending ICD freeze
```

### Designer/uArch scope later

```text
FIFO implementation and depth
Internal pipeline stages
Aurora wrapper decomposition
Error counter implementation
Clock mux implementation details
Reset sequencing implementation
Exact .sv file partitioning
```

---

## 15. Key mental model

The clean separation is:

```text
Optical link = physical fiber path
SERDES      = high-speed serial electrical conversion in the FPGA transceiver
Aurora      = AMD/Xilinx link protocol over SERDES
OBG         = BPMS spectral-bin payload/framing concept
obg_link    = SFU FPGA block that terminates the OBG-over-Aurora transport
CDR         = mechanism that recovers timing from the incoming serial stream
```

And the key architecture point is:

```text
`obg_link` is not just a data input block.
It is a link, framing, CDC, status, and clock-recovery boundary block.
```

That is why `obg_link` is the right abstraction for the FAD and Module Spec layer.
