# Verification Plan: INCORE_CLINT Register Verification (UVM RAL)

| | |
|---|---|
| **Document Title** | Verification Plan – INCORE_CLINT Register Verification |
| **DUT / IP** | INCORE_CLINT |
| **IP Version** | 1.0 |
| **Vendor** | example.org |
| **Library** | mylibrary |
| **Source of Truth** | `INCORE_CLINT.regman` (rtl_1p0_release_regman) |
| **Verification Method** | UVM Register Abstraction Layer (RAL) |
| **Document Status** | Draft – Generated strictly from RegMan content |
| **Date** | 2026-07-14 |

---

## 1. Document Overview

### 1.1 Purpose

This document defines the Verification Plan (VPlan) for the register space of the **INCORE_CLINT** peripheral, to be verified using the UVM Register Abstraction Layer (RAL). The plan defines verification objectives, strategy, planned features, test plan, functional coverage, and traceability for all registers and fields exposed via the `APB4_interface0` bus interface, as declared in the RegMan source file.

### 1.2 Scope

This VPlan covers **only** the register content explicitly defined in the file:

```
/data/work_area/shreya_sb/claude_code/Regression_26/Rjnsoc/system_reg_map/rjn_soc_reg_map/rtl_1p0_release_regman/INCORE_CLINT.regman
```

No other files (RTL, testbench, UVM regmodel `.sv`, `.rdl`, `.vc`, VHDL sources, or prior documentation) were read, referenced, or used to infer any information in this document, per verification-plan authoring instructions. Any behavior, timing, hardware interaction, or protocol detail not explicitly present in the RegMan is marked **"Information Required"** rather than assumed.

In scope:
- Registers `msip`, `mtimecmp`, `mtime` and their bitfields, as declared under bus interface `APB4_interface0` / address map `clint0_regfile_mmap`.
- Frontdoor (APB4) and backdoor (RAL peek/poke) register access verification.
- Reset value, access policy, and negative-access verification based solely on RegMan-declared attributes.

Out of scope / not determinable from the RegMan:
- Functional/interrupt behavior of the `MSIP`/`mip` CSR interaction beyond the textual field description.
- Timer tick / hardware counter behavior of `mtime` (no such attribute is declared in the RegMan — see §4.3 and §1.4 clarifications).
- Security/privilege fault behavior (no behavioral definition provided for `non_secure` / `privileged` flags — see §1.4).
- System-level integration, interrupt propagation to the hart, or multi-hart addressing (RegMan defines a single instance of each register; `array_size = 1` for all).

### 1.3 References

| Ref | Document | Notes |
|---|---|---|
| R1 | `INCORE_CLINT.regman` (rtl_1p0_release_regman, schema/tool version `15.0.0`) | Sole source for this VPlan |
| R2 | UVM 1.2 Register Layer (RAL) User Guide | Verification methodology reference (external, not IP-specific) |
| R3 | AMBA APB4 Protocol Specification | Bus protocol reference implied by `protocol: "APB4"` in RegMan (external standard, not read as part of this task) |

> **Information Required:** The RegMan top-level `"description"` field for INCORE_CLINT is empty. No block-level functional description, application note, or architectural reference is available beyond what is stated per-register.

### 1.4 Clarifications Applied (per stakeholder confirmation)

The following interpretations were explicitly confirmed with the requester prior to drafting this VPlan, because the RegMan does not itself define them:

| Item | RegMan Evidence | Interpretation Applied in this VPlan |
|---|---|---|
| `hw_access: "Default"` on `msip` (bit 0), `mtimecmp`, `mtime` fields (vs. explicit `"RO"` on the reserved field) | No enumerated definition of `"Default"` is given in the file | Treated as **plain RW with no autonomous hardware update** — hardware does not asynchronously modify the field; value is fully determined by software read/write. Standard RAL mirror prediction applies. |
| Volatility of `mtime` / `mtimecmp` | No `volatile` attribute, no counter/tick property, no hardware-update description anywhere in the RegMan | Modeled **strictly as declared**: ordinary non-volatile RW registers. Value changes only via software write. (Note: this is a deliberate as-documented modeling choice — see §4.3 caveat.) |
| `non_secure: false` / `privileged: false` on all 3 registers | Booleans present, but no behavioral/fault definition for a disallowed access is declared | **No dedicated security/privilege-mode verification** is included. Attribute values are recorded in the register summary only (§2.1) and are **not** exercised as pass/fail criteria. |

These three interpretations should be re-confirmed against the RTL/architecture specification before sign-off, as they materially affect test predication.

---

## 2. DUT Register Overview

### 2.1 Register Summary

Bus Interface: `APB4_interface0` — Protocol: **APB4**, Address Width: **32 bits**, Data Width: **64 bits**, Error Response: **enabled** (`error_response: true`)
Address Map: `clint0_regfile_mmap` — Address Unit: **8 bits (byte-addressed)**, Data Width: **64 bits**, Map Size: **0x10000 (64 KB)**

| # | Register | Offset | Width (bits) | Array Size | SW Access | Reset Value | Description (verbatim from RegMan) |
|---|---|---|---|---|---|---|---|
| 1 | `msip` | `0x0` | 32 | 1 | RW | `0x00000000` | This register generates machine mode software interrupts when set. |
| 2 | `mtimecmp` | `0x4000` | 64 | 1 | RW | `0x0000000000000000` | This register holds the compare value for the timer. |
| 3 | `mtime` | `0xBFF8` | 64 | 1 | RW | `0x0000000000000000` | Provides the current timer value. |

Common per-register attributes (identical across all 3 registers, per RegMan):
`non_secure = false`, `privileged = false`, `memory = false`, `visible_we = false`, `parity_check = false`, `presence_condition = "" (unconditionally present)`, `user_defined_properties = {}`.

> **Information Required:** The address map declares a total window size of `0x10000` (64 KB), but only 3 registers are defined, occupying offsets `0x0–0x3` (msip), `0x4000–0x4007` (mtimecmp) and `0xBFF8–0xBFFF` (mtime). The remaining offsets (e.g., `0x4–0x3FFF`, `0x4008–0xBFF7`, `0xC000–0xFFFF`) are unmapped/reserved. The RegMan does not state the expected access behavior for these unmapped offsets (e.g., whether reads return `0`, return unpredictable data, or whether `error_response: true` guarantees a bus error/SLVERR for every unmapped access in this window). This must be clarified before finalizing negative-test pass/fail criteria (see §4.5, §6.9).

### 2.2 Register Map (Address Layout)

```
Offset        Register     Width    Notes
0x0000        msip         32-bit   Base of map
0x0004–0x3FFF (unmapped)            Information Required — access behavior undefined
0x4000        mtimecmp     64-bit
0x4008–0xBFF7 (unmapped)            Information Required — access behavior undefined
0xBFF8        mtime        64-bit   Ends at 0xBFFF
0xC000–0xFFFF (unmapped)            Information Required — access behavior undefined
                                    Map size = 0x10000 (end of window)
```

### 2.3 Field Summary

| Register | Field | Bit Range | Reset | SW Access | HW Access | Read Behavior | Write Behavior | Description |
|---|---|---|---|---|---|---|---|---|
| `msip` | `msip_reserved_0` | [31:1] | `0b0` | RO | RO | R | NA | Reserved field |
| `msip` | `msip` | [0:0] | `0b0` | RW | Default (treated as plain RW, §1.4) | R | W | Machine-mode software interrupts are generated by writing to the memory-mapped control register `msip`. The `msip` register is a 32-bit wide WARL register where the upper 31 bits are tied to 0. The least significant bit can be used to drive the `MSIP` bit of the `mip` CSR of a RISC-V hart. Other bits in the `msip` register are hardwired to zero. On reset, the `msip` register is cleared to zero. |
| `mtimecmp` | `mtimecmp` | [63:0] | `0b0` | RW | Default (treated as plain RW, §1.4) | R | W | *(Information Required — field-level description is empty in RegMan; register-level description applies: "This register holds the compare value for the timer.")* |
| `mtime` | `mtime` | [63:0] | `0b0` | RW | Default (treated as plain RW, §1.4) | R | W | *(Information Required — field-level description is empty in RegMan; register-level description applies: "Provides the current timer value.")* |

All fields: `external_reset_source = ""` (no distinct external reset source defined — default/single reset domain applies), `write_enable = ""` (no conditional write-enable gating declared), `presence_condition = ""` (unconditionally present), `enumerated_values = []` (no enumerated legal values declared), `user_defined_properties = {}`.

---

## 3. Verification Objectives

Based solely on the RegMan content, the verification of INCORE_CLINT registers shall demonstrate that:

1. **OBJ-1**: Every declared register (`msip`, `mtimecmp`, `mtime`) is correctly accessible at its declared offset via the APB4 frontdoor interface, with the declared data width (64-bit data bus, per `data_width: 64`).
2. **OBJ-2**: Every field behaves per its declared `sw_access` / `read_behavior` / `write_behavior` (RO for `msip_reserved_0`; RW for `msip`, `mtimecmp`, `mtime` functional fields).
3. **OBJ-3**: All registers/fields reset to their declared reset value (`0b0` in all cases) following a DUT reset.
4. **OBJ-4**: The RAL model's mirror value tracks the DUT register value correctly across frontdoor writes, reads, and backdoor peek/poke operations (mirror/desired/actual consistency).
5. **OBJ-5**: The reserved field `msip_reserved_0` [31:1] is hardwired RO and cannot be modified by software writes (WARL/tied-to-0 behavior as described).
6. **OBJ-6**: Accesses that violate the declared access policy (e.g., writes to the RO reserved field) do not alter DUT state, consistent with declared `sw_access`.
7. **OBJ-7 (Information Required)**: Access to unmapped offsets within the `0x10000` address window produces a well-defined response — pending clarification per §2.1.

---

## 4. Verification Strategy

### 4.1 Frontdoor Verification Strategy

- All 3 registers shall be accessed via the UVM RAL frontdoor path over the `APB4_interface0` bus (APB4 protocol, 32-bit address, 64-bit data), exercising the RAL-generated `reg_block` for INCORE_CLINT.
- Each register/field shall be written and read back through the RAL `write()` / `read()` sequence API, and results checked against the RAL mirror.
- Frontdoor tests shall use the declared register widths: 32-bit access for `msip`, 64-bit access for `mtimecmp` and `mtime`.

### 4.2 Backdoor Verification Strategy

- Each register shall be verified via RAL `peek()` / `poke()` where a backdoor HDL path is available, to confirm the RAL model's backdoor access is consistent with the frontdoor-visible value.
- Backdoor poke of the reserved field `msip_reserved_0` is used to confirm the field is truly hardwired (i.e., poking it and reading back via frontdoor should reflect RTL-level tie-off behavior) — **Information Required**: the RegMan states the field is `RO`/hardwired but does not state whether the underlying flop exists at all (tied to `0` in logic) or exists but is not software-writable; this affects whether a backdoor poke is expected to "stick." To be confirmed against RTL prior to writing this specific test.

### 4.3 Register Reset Verification

- All registers/fields reset to `0b0` per the RegMan `reset` attribute. Reset verification shall confirm:
  - `msip` == `0x00000000` after reset (both bits [0:0] and reserved [31:1]).
  - `mtimecmp` == `0x0000000000000000` after reset.
  - `mtime` == `0x0000000000000000` after reset.
- No distinct `external_reset_source` is declared for any field (all are empty strings) — reset verification shall use the single/default reset domain only. No multi-domain or asynchronous-external-reset test is planned, as none is declared.
- **Caveat (per §1.4):** `mtime`/`mtimecmp` are modeled strictly as documented (non-volatile RW) for this VPlan. If subsequent RTL/architecture review reveals `mtime` free-runs as a hardware timer (as is typical for RISC-V CLINT implementations, but not stated in this RegMan), the reset-verification and mirror-tracking tests in §6 will need to change to volatile-register techniques (e.g., read without mirror compare, or bounded/monotonic-value checks instead of exact-match). This VPlan does **not** currently include volatile-register handling, per explicit confirmation in §1.4.

### 4.4 Register Access Policy Verification

- Verify declared `sw_access` policies exactly as specified:
  - `msip.msip_reserved_0` — **RO**: writes of any value are ignored; reads always return the current field value (expected `0` per hardwired/reserved description).
  - `msip.msip` — **RW**: writes update the field; reads return the last written value.
  - `mtimecmp.mtimecmp` — **RW**: writes update the field; reads return the last written value.
  - `mtime.mtime` — **RW**: writes update the field; reads return the last written value.
- Register-level `sw_access` for all 3 registers is `RW`, consistent with their constituent fields.
- Per §1.4, `non_secure`/`privileged` attribute values (`false` for all registers) are recorded for completeness (§2.1) but are **not** exercised as distinct access-policy test vectors, since the RegMan defines no fault/violation behavior to check against.

### 4.5 Negative Testing Strategy

- Writes to the RO reserved field `msip_reserved_0` [31:1] shall be attempted with all-ones and randomized patterns; the field must not change from its reset/current value, and adjacent field `msip` [0:0] must remain unaffected (bit-level isolation check).
- Byte-enable / partial-access negative checks are **Information Required** — RegMan does not declare byte-strobe or sub-word access restrictions for any register.
- Out-of-range / unmapped-offset access testing (e.g., offset `0x8`, `0x5000`, `0xD000`) is **Information Required** pending clarification of expected behavior for the unmapped regions identified in §2.1/§2.2. `error_response: true` at the bus-interface level suggests the APB4 interconnect supports error responses, but the RegMan does not state whether every unmapped offset within the 64 KB window triggers this response.
- Illegal access-width negative tests (e.g., 64-bit access to `msip`, which is declared 32-bit) shall be planned generically per the declared register widths; the RegMan does not define specific illegal-width fault behavior beyond the general `error_response: true` bus capability.

---

## 5. Verification Features

Feature IDs are of the form `FT-<REGISTER>-<NNN>`.

### 5.1 Register: `msip` (offset `0x0`, 32-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-MSIP-001 | Frontdoor write/read of full register | RAL frontdoor `write()`/`read()` via APB4 | Read-back value matches last written value in bits [0:0]; bits [31:1] always read `0` |
| FT-MSIP-002 | Reserved field `msip_reserved_0` [31:1] is RO/hardwired | RAL frontdoor write of non-zero pattern to [31:1], then read | Field reads back `0` regardless of write attempt (WARL, tied to 0 per description) |
| FT-MSIP-003 | Functional bit `msip` [0:0] RW behavior | RAL frontdoor write `1`, read back; write `0`, read back | Bit reflects last written value (`1` or `0`) |
| FT-MSIP-004 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000000` |
| FT-MSIP-005 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` on bit [0:0] | Poked value observable via subsequent frontdoor read; peek matches last frontdoor-written value |
| FT-MSIP-006 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value; no `UVM_ERROR` on compare |
| FT-MSIP-007 | Bit-isolation negative test | Write reserved bits with all-ones while `msip` bit = 0; read back | Reserved bits = 0, `msip` bit unaffected (remains 0) |

### 5.2 Register: `mtimecmp` (offset `0x4000`, 64-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-MTIMECMP-001 | Frontdoor write/read of full 64-bit register | RAL frontdoor `write()`/`read()` via APB4 (64-bit data bus) | Read-back value equals last written 64-bit value |
| FT-MTIMECMP-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x0000000000000000` |
| FT-MTIMECMP-003 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read; peek matches last written value |
| FT-MTIMECMP-004 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-MTIMECMP-005 | Boundary value test | Write `0x0`, `0xFFFFFFFFFFFFFFFF`, alternating bit patterns (`0x5555...`, `0xAAAA...`) | All patterns read back exactly as written (full 64-bit RW, no masked bits declared) |
| FT-MTIMECMP-006 (Information Required) | Interaction with `mtime` compare/interrupt logic | Not determinable from RegMan | RegMan does not describe the comparator/interrupt-generation behavior between `mtime` and `mtimecmp`; only register-level RW access is defined. Flagged for architecture-spec confirmation. |

### 5.3 Register: `mtime` (offset `0xBFF8`, 64-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-MTIME-001 | Frontdoor write/read of full 64-bit register | RAL frontdoor `write()`/`read()` via APB4 (64-bit data bus) | Read-back value equals last written 64-bit value (per §1.4, modeled as non-volatile RW) |
| FT-MTIME-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x0000000000000000` |
| FT-MTIME-003 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read; peek matches last written value |
| FT-MTIME-004 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-MTIME-005 | Boundary value test | Write `0x0`, `0xFFFFFFFFFFFFFFFF`, alternating bit patterns | All patterns read back exactly as written |
| FT-MTIME-006 (Information Required) | Hardware auto-increment / free-running counter behavior | Not determinable from RegMan | No volatile/tick/counter attribute is declared (see §1.4 caveat); if RTL implements a free-running timer, this feature and its test method must be revised to volatile-register techniques. |

### 5.4 Address Map Features

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-MAP-001 | Correct address decode for all 3 registers | Frontdoor access at each declared offset (`0x0`, `0x4000`, `0xBFF8`) | Access reaches the intended register only; no aliasing to adjacent/unmapped space |
| FT-MAP-002 (Information Required) | Unmapped-offset access behavior | Frontdoor access to offsets outside the 3 declared registers, within the `0x10000` window | RegMan does not define expected response (error vs. default read); pending clarification |
| FT-MAP-003 | Bus error-response capability | Negative-path frontdoor access expected to trigger bus-level error | `error_response: true` declared at bus-interface level — exact triggering conditions beyond unmapped access are not specified; scope of this feature limited to what RegMan declares |

---

## 6. Test Plan

| Test ID | Test Name | Applicable Register(s) | Category | Description |
|---|---|---|---|---|
| TC-001 | `msip_frontdoor_rw` | msip | Frontdoor R/W | Write/read msip bit via APB4 frontdoor; verify mirror match |
| TC-002 | `mtimecmp_frontdoor_rw` | mtimecmp | Frontdoor R/W | Write/read full 64-bit mtimecmp via APB4 frontdoor |
| TC-003 | `mtime_frontdoor_rw` | mtime | Frontdoor R/W | Write/read full 64-bit mtime via APB4 frontdoor (per §4.3 non-volatile modeling) |
| TC-004 | `msip_backdoor_peek_poke` | msip | Backdoor | RAL `peek()`/`poke()` on msip bit; cross-check with frontdoor |
| TC-005 | `mtimecmp_backdoor_peek_poke` | mtimecmp | Backdoor | RAL `peek()`/`poke()` on mtimecmp; cross-check with frontdoor |
| TC-006 | `mtime_backdoor_peek_poke` | mtime | Backdoor | RAL `peek()`/`poke()` on mtime; cross-check with frontdoor |
| TC-007 | `all_regs_reset_value_check` | msip, mtimecmp, mtime | Reset | Apply reset; verify all registers/fields equal declared reset value (`0`) |
| TC-008 | `msip_reserved_field_ro_check` | msip | Access Permission (RO) | Attempt write to `msip_reserved_0`; verify no change, reads as 0 |
| TC-009 | `msip_field_rw_check` | msip | Access Permission (RW) | Verify `msip` bit is fully RW per declared access |
| TC-010 | `mtimecmp_field_rw_check` | mtimecmp | Access Permission (RW) | Verify `mtimecmp` field is fully RW per declared access |
| TC-011 | `mtime_field_rw_check` | mtime | Access Permission (RW) | Verify `mtime` field is fully RW per declared access |
| TC-012 | `all_regs_mirror_update_check` | msip, mtimecmp, mtime | Mirror/Update | RAL `mirror()`/`update()` calls after writes; confirm no mismatch reported |
| TC-013 | `ral_built_in_hdl_reg_access_seq` | msip, mtimecmp, mtime | Random Access | Standard UVM RAL built-in sequence (`uvm_reg_hw_reset_seq`, `uvm_reg_bit_bash_seq`, `uvm_reg_access_seq`) applied to all registers |
| TC-014 | `mtimecmp_mtime_boundary_values` | mtimecmp, mtime | Boundary | Write `0x0`, all-ones, walking-1/walking-0 patterns; verify exact read-back |
| TC-015 | `msip_bit_isolation_negative` | msip | Negative | Write reserved bits; verify functional bit unaffected and reserved bits remain 0 |
| TC-016 (Information Required) | `unmapped_offset_access` | Address map | Negative | Access offsets outside declared registers within `0x10000` window — expected response pending clarification (§2.1, §4.5) |
| TC-017 (Information Required) | `illegal_access_width` | msip, mtimecmp, mtime | Negative | Access with width other than declared (e.g., 64-bit access to 32-bit `msip`) — exact fault behavior not specified in RegMan beyond generic `error_response: true` |

---

## 7. Functional Coverage Plan

| Coverage Group | Coverpoints | Source |
|---|---|---|
| `cg_register_access` | Per-register: read hit, write hit, for `msip`, `mtimecmp`, `mtime` | Register list, §2.1 |
| `cg_msip_field_values` | `msip` bit = 0, 1 (bin per value); reserved field always sampled as 0 | Field declaration, §2.3 |
| `cg_mtimecmp_values` | Value bins: `0x0`, all-ones, min/max boundary, random mid-range | Boundary test plan, §6 TC-014 |
| `cg_mtime_values` | Value bins: `0x0`, all-ones, min/max boundary, random mid-range | Boundary test plan, §6 TC-014 |
| `cg_access_type` | Frontdoor read, frontdoor write, backdoor peek, backdoor poke — crossed with each register | §4.1, §4.2 |
| `cg_reset_coverage` | Reset applied + register read = expected reset value, per register | §4.3 |
| `cg_access_policy` | RO-field write-attempt (no effect) vs. RW-field write (effective), per field | §4.4 |
| `cg_unmapped_access` (Information Required) | Access-outside-map coverpoint — bin definition depends on clarified expected behavior (§2.1) | Pending |

> No RegMan-declared `enumerated_values` exist for any field (all are `[]`), so no enumerated-value coverage bins are defined; only reset/boundary/random value bins are applicable.

---

## 8. Requirements Traceability Matrix (RTM)

| Req ID | Requirement | Register/Field | Planned Test(s) | Verification Method |
|---|---|---|---|---|
| REQ-01 | msip register accessible at offset 0x0, 32-bit | msip | TC-001, TC-007 | Frontdoor (RAL) |
| REQ-02 | msip bit [0] is RW, drives MSIP/mip interrupt bit | msip.msip | TC-001, TC-009 | Frontdoor, Access Policy |
| REQ-03 | msip bits [31:1] hardwired RO, always 0 | msip.msip_reserved_0 | TC-008, TC-015 | Frontdoor negative, bit-isolation |
| REQ-04 | msip resets to 0x00000000 | msip | TC-007 | Reset |
| REQ-05 | mtimecmp register accessible at offset 0x4000, 64-bit RW | mtimecmp | TC-002, TC-007, TC-010 | Frontdoor, Reset, Access Policy |
| REQ-06 | mtimecmp resets to 0x0 | mtimecmp | TC-007 | Reset |
| REQ-07 | mtimecmp full-width boundary values are read/write consistent | mtimecmp.mtimecmp | TC-014 | Boundary |
| REQ-08 | mtime register accessible at offset 0xBFF8, 64-bit RW | mtime | TC-003, TC-007, TC-011 | Frontdoor, Reset, Access Policy |
| REQ-09 | mtime resets to 0x0 | mtime | TC-007 | Reset |
| REQ-10 | mtime full-width boundary values are read/write consistent | mtime.mtime | TC-014 | Boundary |
| REQ-11 | RAL mirror tracks DUT state across all registers | msip, mtimecmp, mtime | TC-012, TC-013 | Mirror/Update |
| REQ-12 | Backdoor peek/poke consistent with frontdoor state | msip, mtimecmp, mtime | TC-004, TC-005, TC-006 | Backdoor |
| REQ-13 (Information Required) | Unmapped address-space access behavior defined | Address map (0x10000 window) | TC-016 | Negative — pending clarification |
| REQ-14 (Information Required) | Illegal access-width fault behavior defined | msip, mtimecmp, mtime | TC-017 | Negative — pending clarification |
| REQ-15 (Information Required) | mtime hardware free-run / auto-increment behavior defined | mtime | FT-MTIME-006 | Not testable from RegMan alone — pending architecture spec |
| REQ-16 (Information Required) | mtimecmp-to-mtime comparator/interrupt behavior defined | mtimecmp, mtime | FT-MTIMECMP-006 | Not testable from RegMan alone — pending architecture spec |

---

## 9. Regression Plan

- **Regression Scope**: All tests in §6 (TC-001 through TC-015) form the baseline regression suite for every code/RTL change affecting INCORE_CLINT, using the RAL model generated from `INCORE_CLINT.regman` (rtl_1p0_release_regman variant).
- **Regression Composition**:
  - Directed tests: TC-001–TC-011, TC-015 (deterministic per-register/per-field checks).
  - Built-in RAL sequences (random/generic): TC-013, run with multiple random seeds per regression cycle.
  - Boundary tests: TC-014, run with fixed corner-case values every regression, plus randomized mid-range values per seed.
- **Pass/Fail Criteria**: No RAL `UVM_ERROR`/`UVM_FATAL` on register mirror mismatch; all declared reset values, access policies, and boundary values match exactly.
- **Exclusions Pending Clarification**: TC-016, TC-017 are **not** included in the baseline regression pass/fail gate until the "Information Required" items in §1.4, §2.1, §4.5 are resolved with the register/architecture owner. They are tracked as open items and should be added once clarified.
- **Seed/Iteration Policy**: Information Required — the RegMan does not define a required regression seed count, iteration count, or coverage closure threshold; this is a testbench/methodology decision outside the scope of the RegMan and should be defined by the verification team.
- **Re-run Triggers**: Any change to `INCORE_CLINT.regman` (offsets, widths, access types, reset values) should trigger regeneration of the RAL model and a full re-run of this regression suite.

---

## Appendix A: Open Items Requiring Clarification (Consolidated)

| # | Item | Section | Impact |
|---|---|---|---|
| 1 | Expected behavior for accesses to unmapped offsets within the `0x10000` address window | §2.1, §4.5, TC-016, REQ-13 | Negative test pass/fail criteria, functional coverage bins |
| 2 | Illegal access-width fault behavior (beyond generic `error_response: true`) | §4.5, TC-017, REQ-14 | Negative test pass/fail criteria |
| 3 | Whether `mtime` free-runs in hardware (RISC-V CLINT convention) vs. is purely software-driven as modeled here | §1.4, §4.3, FT-MTIME-006, REQ-15 | Fundamental test/predication strategy for `mtime` |
| 4 | Comparator/interrupt-generation relationship between `mtimecmp` and `mtime` | FT-MTIMECMP-006, REQ-16 | Whether cross-register functional tests are needed beyond independent register access |
| 5 | Backdoor writability of the tied-off reserved field `msip_reserved_0` | §4.2 | Whether TC-004's backdoor poke on reserved bits is a valid/meaningful test |
| 6 | Regression seed count / coverage closure threshold | §9 | Regression sizing and exit criteria |

This VPlan should be reviewed against the RTL/UVM regmodel and the architecture specification to resolve the items above prior to test implementation sign-off.
