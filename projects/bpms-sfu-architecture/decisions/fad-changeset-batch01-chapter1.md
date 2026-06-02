# Change-set — FAD batch 01 (Chapter 1 review)

**For:** `repo_operator` (Claude Code), repo `bpms-sfu-fpga-design`
**Author of decisions:** Marco Pausini · **Drafted by:** fpga_arch · **Date:** 2026-05-28
**Scope:** FAD §1 review feedback (Chapter 1 only). Each change is anchored exact-match.
**Apply method:** `str_replace` / exact find-replace. Anchors are byte-exact; do not normalise whitespace.
**Pre-flight:** confirm `arch/fad/FAD.md` working tree is clean and matches the reviewed v0.7 (the byte anchors below were taken from it). If the path differs in `CLAUDE.md`, retarget but keep anchors.

Decisions locked: D1 invariants at **§1.3** (+ 4 methodology ref updates); D2 ADR-0003 `owner: Marco`; D3 parent-doc version **v03.01**; D-opt **no** R4 marker (FAD-ARCH-01 mgmt wording left untouched, deferred to §2.3); `tle_compute` rename deferred.

Suggested commit grouping: one commit for C1 (`docs(fad): add §1.3 architecture invariants`), one for C2–C8 (`docs(fad): chapter-1 review fixes`), one for C9–C12 (`docs(methodology): retarget invariant refs §1.5→§1.3`). C13 optional.

---

## C1 — Insert §1.3 Architecture invariants  ·  review R1 (blocker) + R7 (gap)  ·  file `arch/fad/FAD.md`

Resolves: missing invariants (G1 blocker) and the §1.2.3→§1.4 numbering gap. Methodology pins invariants to a fixed §; C9–C12 retarget it to §1.3.

**FIND:**
```
3. If no such source exists, the proposal is rejected by default.


### 1.4 Key decisions
```

**REPLACE:**
```
3. If no such source exists, the proposal is rejected by default.


### 1.3 Architecture invariants

Architecture invariants are cross-cutting constraints that hold across the entire SFU PL design. No Module Spec, ICD, ADR, or any of §2–§6 may violate them; any proposal that would requires an ADR citing a system-level source change, per the boundary-crossing procedure (§1.2.3). Each invariant traces to a parent-source section; inferred elements carry an explicit marker.

| ID | Architecture invariant | Source |
|---|---|---|
| INV-01 | **SFU owns band-level processing only.** Per-AxC Beam Gain, Beam Doppler, and beam delay are DCU-owned and complete before OBG ingress. | SFU-001 §4, §4.4, §5; ARCH-001 §5.6 |
| INV-02 | **No per-beam function may be introduced inside the SFU PL** without an ADR and a parent-source change (enforces INV-01; see §1.2.3). | SFU-001 §4.4, §5; FAD §1.2.3 |
| INV-03 | **SFU receives spectral-bin traffic over OBG and does not terminate CPRI** (CPRI termination is DCU-owned). | SFU-001 §4.2; ARCH-001 §5.2 |
| INV-04 | **OBG Q0 is four independent, full-duplex Aurora SERDES lanes** (one per OBG link) through the optical matrix; the same physical lanes carry DL and UL. | SFU-001 §6.1, §7.5 |
| INV-05 | **OBG inter-lane skew is compensated** (per-lane programmable delay) before DL bins are selected and combined for filter-bank synthesis. | SFU-001 §4.2, §6.2 |
| INV-06 | **SFU filter-bank processing is two sub-bands** (A = lower 1,024 MHz, B = upper 1,024 MHz); sub-band replication is instance parallelism, not separate functional ownership (FAD-ARCH-02). | SFU-001 §4.2, §5; ARCH-001 §5.2 |
| INV-07 | **DL and UL pipelines are independent** except for the shared `rf_port_sel` and the ADC/DAC physical converters. | SFU-001 §5 |
| INV-08 | **DCU owns FPGA-satellite bin-grid alignment via its dual ASIC/FPGA filter bank** (ARCH §7.3 Option A); the SFU receives natively-aligned OBG bins and is unaware of the per-OBG choice. | ARCH-001 §7.3, §5.2; SFU-001 §6.3 |
| INV-09 | **One SFU serves one satellite type (ASIC or FPGA) at a time;** orientation is set at commissioning, stored in NV config; changing it requires planned re-commissioning. Not satellite-agnostic at runtime. | SFU-001 §4.1, §4.2; ARCH-001 §7.3 |
| INV-10 | **RF port selection switches TX and RX as a paired RF path** (UTC-scheduled, NV-stored); no independent TX-only/RX-only operational switch absent a parent-source change. | SFU-001 §6.7, §7.1 |
| INV-11 | **Operational Band Doppler is Local Autonomous and 1PPS-aligned:** the SFU latches a TLE-derived NCO word at the 1PPS tick; Manager SW never writes live Doppler values. | SFU-001 §4.4, §6.6; ARCH-001 §15.3 |
| INV-12 | **UTC-Scheduled parameters are pre-loaded with a future UTC timestamp, committed at the UTC-counter match (1PPS-derived tick), and confirmed by telemetry** to Manager SW. | SFU-001 §4.4, §6.5; ARCH-001 §15.2 |
| INV-13 | **UTC-Scheduled and Local Autonomous are distinct apply classes** and must not be collapsed in wording or mechanism tables. | SFU-001 §4.4; ARCH-001 §15.2–§15.3 |
| INV-14 | **Primary SFU timing derives from the CDR-recovered clock of the commissioning-selected timing-master OBG DL lane; SFU cards in a G-SFU group share no inter-card data path.** `[INFERRED: inter-card timing alignment uses an external reference + device-internal phase trim, not a shared clock net — SFU-001 §11.5 is STUB (session 2); confirm at §11 release.]` | SFU-001 §6.1, §7.5, §4.3 |


### 1.4 Key decisions
```

---

## C2 — §1.2.1 UTC-apply row mechanism  ·  review R2 (high)  ·  file `arch/fad/FAD.md`

Collapsed two apply classes into one. New wording matches the §2.3 apply table and INV-12/INV-13; avoids "UTC tick" (reads as 1PPS).

**FIND:**
```
| UTC-scheduled parameter apply (shadow→live) | Per parameter | Local autonomous (UTC tick) | SFU-001 §17.1, ARCH-001 §15.2 |
```

**REPLACE:**
```
| UTC-scheduled parameter apply (shadow→live) | Per parameter | UTC-scheduled (shadow→live commit at UTC-counter match) | SFU-001 §17.1, ARCH-001 §15.2 |
```

---

## C3 — §1.2.1 Band Doppler PL-side row  ·  review R3 (high)  ·  file `arch/fad/FAD.md`

PL does **not** compute Doppler from TLE — PS runs SGP4 and produces the NCO word; PL latches `next→live` at 1PPS. Renamed to reflect that; citation retargeted off the stubbed SFU §11.3 to §6.6 + ARCH §15.3. Consistent with §1.2.2 (SGP4/NCO-word generation = PS, out of scope).

**FIND:**
```
| Local Doppler computation from TLE — PL side (NCO load at 1PPS) | Per sub-band per direction | Local autonomous | SFU-001 §6.6, §11.3 |
```

**REPLACE:**
```
| 1PPS-aligned Band Doppler NCO-word load (next→live latch) — PL side | Per sub-band per direction | Local autonomous (1PPS latch) | SFU-001 §6.6, ARCH-001 §15.3 |
```

---

## C4 — §1.2.2 TLE/SGP4 row TBD marker  ·  review R5 (framework)  ·  file `arch/fad/FAD.md`

**FIND:**
```
| TLE/SGP4 ephemeris propagation (NCO frequency word generation) | PS software (separate repo, TBD) | This FAD §1.4.1 (FAD-ARCH-01) |
```

**REPLACE:**
```
| TLE/SGP4 ephemeris propagation (NCO frequency word generation) | PS software [TBD: PS-software repo, owner: SW Lead] | This FAD §1.4.1 (FAD-ARCH-01) |
```

---

## C5 — §1.2.2 management-protocol row TBD marker  ·  review R5 (framework)  ·  file `arch/fad/FAD.md`

**FIND:**
```
| Management protocol stack (REST/SNMP/TBD) | PS software | This FAD §1.4.1 (FAD-ARCH-01) |
```

**REPLACE:**
```
| Management protocol stack [TBD: protocol (REST/SNMP/other), owner: SW Lead, ref OI-S07] | PS software | This FAD §1.4.1 (FAD-ARCH-01) |
```

---

## C6 — FAD-ARCH-01 ADR-0003 owner  ·  review R5 + D2  ·  file `arch/fad/FAD.md`

Minimal anchor — only the owner field changes. The mgmt-plane wording (`AXI bridge`, `reg_bus`) is intentionally left untouched (D-opt: R4 deferred to §2.3).

**FIND:**
```
[TBD: ADR-0003 PS-PL partitioning, owner: TBD]
```

**REPLACE:**
```
[TBD: ADR-0003 PS-PL partitioning, owner: Marco]
```

---

## C7 — §1.5 parent-doc table: add Status/Date, stamp v03.01  ·  review R6 + D3  ·  file `arch/fad/FAD.md`

**FIND:**
```
| Ref | Document | ID | Version |
|---|---|---|---|
| [P1] | BPMS 1.0 System Architecture Document | BPMS-1.0-ARCH-001 |  |
| [P2] | BPMS 1.0 SFU Architecture Document | BPMS-1.0-SFU-001 |  |
| [P3] | Filter Bank Design Specification — Device A/B Block 2 | (Fryderyk Fijalkowski, AST SpaceMobile) | 0.1 |
```

**REPLACE:**
```
| Ref | Document | ID | Version | Status | Date |
|---|---|---|---|---|---|
| [P1] | BPMS 1.0 System Architecture Document | BPMS-1.0-ARCH-001 | v03.01 | Draft (HTML-primary) | 2026-05 |
| [P2] | BPMS 1.0 SFU Architecture Document | BPMS-1.0-SFU-001 | v03.01 | Draft (HTML-primary) | 2026-05 |
| [P3] | Filter Bank Design Specification — Device A/B Block 2 | (Fryderyk Fijalkowski, AST SpaceMobile) | 0.1 | Draft | — |
```

---

## C8 — §14.2 References: align [P1]/[P2] to v03.01  ·  review R6 + D3 (appendix consistency)  ·  file `arch/fad/FAD.md`

The references appendix still carries legacy ARCH v2.4 / SFU v1.6; align to §1.5 so the FAD is internally consistent on parent-doc versions.

**FIND:**
```
| [P1] | BPMS 1.0 System Architecture Document (BPMS-1.0-ARCH-001 v2.4) |
| [P2] | BPMS 1.0 SFU Architecture Document (BPMS-1.0-SFU-001 v1.6) |
```

**REPLACE:**
```
| [P1] | BPMS 1.0 System Architecture Document (BPMS-1.0-ARCH-001 v03.01) |
| [P2] | BPMS 1.0 SFU Architecture Document (BPMS-1.0-SFU-001 v03.01) |
```

---

## C9 — retarget invariant ref  ·  D1  ·  file `docs/methodology/02-architect-workflow.md`

**FIND:**
```
The FAD §1.5 defines architecture invariants — constraints that apply across the entire
```

**REPLACE:**
```
The FAD §1.3 defines architecture invariants — constraints that apply across the entire
```

---

## C10 — retarget invariant ref (G1 criterion)  ·  D1  ·  file `docs/methodology/05-signoff-criteria.md`

**FIND:**
```
- Architecture invariants (§1.5) are stated; no other section violates them
```

**REPLACE:**
```
- Architecture invariants (§1.3) are stated; no other section violates them
```

---

## C11 — retarget invariant ref (ICD sign-off)  ·  D1  ·  file `docs/methodology/05-signoff-criteria.md`

**FIND:**
```
- Architecture invariants (§1.5) stated and not violated by §2–§6
```

**REPLACE:**
```
- Architecture invariants (§1.3) stated and not violated by §2–§6
```

---

## C12 — retarget invariant ref  ·  D1  ·  file `docs/methodology/05-signoff-criteria.md`

**FIND:**
```
- Architecture invariants (FAD §1.5) respected
```

**REPLACE:**
```
- Architecture invariants (FAD §1.3) respected
```

---

## C13 — OPTIONAL housekeeping: version + change-log  ·  file `arch/fad/FAD.md`

Apply only if Marco wants the version bumped this batch (default 0.7 → 0.7.1; switch to 0.8 if preferred). Marco may rewrite the change-log prose.

**FIND:**
```
version: 0.7
```
**REPLACE:**
```
version: 0.7.1
```

**FIND:**
```
> **Version:** 0.7 · **Date:** 2026-05-27 · **Status:** in_review · **Author:** Marco Pausini  
```
**REPLACE:**
```
> **Version:** 0.7.1 · **Date:** 2026-05-28 · **Status:** in_review · **Author:** Marco Pausini  
```

Add a change-log row immediately under the `| 0.6 | ...` row in §15:
```
| 0.7.1 | 2026-05-28 | Marco Pausini | **Chapter-1 review fixes (pre-G1).** §1.3 Architecture invariants added (INV-01..INV-14; INV-07/INV-15 from the review list dropped as notation/self-consistency rather than invariants; DL/UL-independence invariant added). §1.2.1 UTC-apply row de-collapsed (UTC-counter-match wording); Band Doppler PL-side row renamed to "1PPS-aligned NCO-word load (next→live latch)" with citation retargeted off stubbed SFU §11.3. §1.2.2 raw TBDs given owned markers. FAD-ARCH-01 ADR-0003 owner set to Marco. §1.5 parent-doc table gains Status/Date and v03.01; §14.2 references aligned to v03.01. Methodology invariant refs retargeted §1.5→§1.3 (02-architect-workflow, 05-signoff-criteria). FAD-ARCH-01/-10 mgmt-plane ownership reconciliation deferred to §2.3 review; `tle_compute` rename deferred. |
```

---

## Post-apply check (the "Check" step)

Run after applying:

```sh
# §1.3 exists, §1.4 still present, contiguous numbering
grep -nE "^### 1\.(2|3|4|5)" arch/fad/FAD.md      # expect 1.2-stub? then 1.3, 1.4, 1.5 in order
grep -c "INV-1[0-4]\|INV-0[1-9]" arch/fad/FAD.md  # expect 14 invariant rows present

# no stale invariant refs left in methodology
grep -rn "invariants (§1.5)\|invariants (FAD §1.5)\|FAD §1.5 defines architecture" docs/methodology/  # expect: no matches

# review fixes landed
grep -n "UTC-counter match" arch/fad/FAD.md                         # C2
grep -n "1PPS-aligned Band Doppler NCO-word load" arch/fad/FAD.md   # C3
grep -n "owner: SW Lead\|owner: Marco" arch/fad/FAD.md              # C4/C5/C6
grep -n "ARCH-001 v03.01\|BPMS-1.0-ARCH-001 | v03.01" arch/fad/FAD.md  # C7/C8

# stale parent versions gone
grep -n "v2.4\|v1.6" arch/fad/FAD.md   # expect: no matches in parent-doc context
```

Render the FAD and eyeball §1.3 table formatting and the §1.5 6-column table.
