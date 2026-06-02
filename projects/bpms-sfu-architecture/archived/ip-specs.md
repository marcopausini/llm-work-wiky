# {{IP_NAME}} — specification

<!-- Audience: DV, firmware, integrator. Defines the contract. No internals. -->

**Status:** draft
**Traces:** `doc/reqs.md`

## Overview

<!-- 1 paragraph. What the block is, from the outside. -->
{{TODO: overview}}

## Top-level ports

<!-- Group rows by clock domain. Use interface names where applicable
     (axi4l_if.slave, axis_if.master, …). Do not enumerate every AXI signal;
     name the modport and link to the interface file.
     Managed block: sync skill rebuilds rows from RTL; prose headers and
     anything outside the markers are preserved. -->

<!-- managed:start regenerate-from=ports -->
### Domain: `sys_clk` / `sys_rst_n`

| Port | Dir | Width / type | Description |
|------|-----|--------------|-------------|
| `sys_clk_in`   | in | 1 | system clock |
| `sys_rst_n_in` | in | 1 | active-low async reset, sync'd internally |
| `s_axi4l`      | slave | `axi4l_if` | register access |

### Domain: `{{TODO: data_clk or _N/A_ if single-domain}}`

| Port | Dir | Width / type | Description |
|------|-----|--------------|-------------|
| {{TODO}} | | | |
<!-- managed:end -->

## Clocks and resets

| Clock | Rate / range | Reset | Reset style | Domain crosses |
|-------|--------------|-------|-------------|----------------|
| {{TODO}} | | | async-assert sync-deassert | list other clocks that cross in |

## Parameters

| Parameter | Default | Valid range | Effect |
|-----------|---------|-------------|--------|
| {{TODO}} | | | |

## Register map

<!-- Authoritative source: one or more YAMLs under regs/. Summarize only;
     do not duplicate. Hierarchical IPs (e.g. a replicated status block)
     ship a separate YAML per register group; list all of them. -->

- YAML(s):
  - `regs/{{IP_NAME}}.yaml`
  - {{TODO: add further regs/*.yaml if hierarchical, or delete this line}}
- Generated header(s): `block_{{IP_NAME}}_t.sv` (via `make regs`)

<!-- managed:start regenerate-from=regs -->
| Offset | Register | Access | Pulse | Purpose |
|--------|----------|--------|-------|---------|
| {{TODO}} | | RW / RO / W1C | W / R / — | |
<!-- managed:end -->

## Operation

<!-- Prose. How the block is used by software / upstream. Configuration
     sequence, steady-state behavior, shutdown. Reference timing diagrams
     below rather than describing waveforms in text. -->

{{TODO: operation}}

## Timing diagrams

<!-- Reference sibling files. Do not inline base64. -->

- `doc/{{IP_NAME}}_<seq>.js` — {{TODO: what it shows}} (WaveDrom)
- _N/A — <reason>_ if the block has no nontrivial protocol.

## Error and status

- **Status bits:** {{TODO or _N/A_}}
- **Interrupts:**  {{TODO or _N/A_}}
- **Illegal-access behavior:** {{TODO or _N/A_}}

## Performance

| Metric | Value | Condition |
|--------|-------|-----------|
| Throughput | {{TODO}} | |
| Latency    | {{TODO}} | ingress→egress, cycles |

## Test plan hooks

<!-- What DV must cover, mapped to requirement IDs from reqs.md. -->

- **R1:** {{TODO: observable behavior / check}}

## Open questions

- {{TODO}}

## References

- `doc/reqs.md`
- `regs/{{IP_NAME}}.yaml`
- {{TODO: related specs}}
 