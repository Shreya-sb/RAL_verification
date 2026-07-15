# Verification Plan: INCORE_PLIC Register Verification (UVM RAL)

| | |
|---|---|
| **Document Title** | Verification Plan – INCORE_PLIC Register Verification |
| **DUT / IP** | INCORE_PLIC |
| **IP Version** | 1.0 |
| **Vendor** | example.org |
| **Library** | mylibrary |
| **Role** | peripheral |
| **Source of Truth** | `INCORE_PLIC.regman` (rtl_1p0_release_regman) |
| **Verification Method** | UVM Register Abstraction Layer (RAL) |
| **Document Status** | Draft – Generated strictly from RegMan content |
| **Date** | 2026-07-15 |

---

## 1. Document Overview

### 1.1 Purpose

This document defines the Verification Plan (VPlan) for the register space of the **INCORE_PLIC** peripheral, to be verified using the UVM Register Abstraction Layer (RAL). The plan defines verification objectives, strategy, planned features, test plan, functional coverage, and traceability for all registers and fields exposed via the `APB4_interface0` bus interface, as declared in the RegMan source file.

### 1.2 Scope

This VPlan covers **only** the register content explicitly defined in the file:

```
/data/work_area/shreya_sb/claude_code/Regression_26/Rjnsoc/system_reg_map/rjn_soc_reg_map/rtl_1p0_release_regman/INCORE_PLIC.regman
```

No other files (RTL, testbench, UVM regmodel `.sv`, `.rdl`, `.vc`, VHDL sources, or prior documentation) were read, referenced, or used to infer any information in this document, per verification-plan authoring instructions. Any behavior, timing, hardware interaction, or configuration detail not explicitly present in the RegMan is marked **"Information Required"** rather than assumed.

In scope:
- Registers `source_priority`, `source_pending`, `target_enables`, `target_threshold`, `target_claim` and their bitfields, as declared under bus interface `APB4_interface0` / address map `plic0_regfile_mmap`.
- Frontdoor (APB4) and backdoor (RAL peek/poke) register access verification.
- Reset value, access policy, and negative-access verification based solely on RegMan-declared attributes.

Out of scope / not determinable from the RegMan:
- The true number of interrupt sources and interrupt targets implemented by the DUT. Every register is declared with `"array_size": 1` in the RegMan, even though field/register descriptions use language such as "the respective interrupt source" (`source_priority`) and "the respective target" (`target_threshold`, `target_claim`) that typically implies multiple instances in a PLIC. Per stakeholder confirmation (§1.4), this is treated as **Information Required** — the VPlan documents and verifies exactly one instance of each register, as literally declared.
- The actual hardware-side read/write interaction implied by `"hw_access": "Default"` on `source_priority`, `target_enables`, `target_threshold`, and `target_claim` fields (no enumerated definition of `"Default"` is given anywhere in the file) — see §1.4.
- Interrupt arbitration/priority-resolution behavior between `source_priority`, `target_threshold`, and `target_claim` (i.e., how the PLIC selects and presents the highest-priority pending, enabled interrupt to a target) — no such behavioral description exists in the RegMan; only per-register static attributes are declared.
- System-level interrupt signal propagation, multi-source/multi-target addressing, or claim/complete side-effect semantics on `target_claim` beyond its declared plain `RW`/`R`/`W` access (see §1.4/§4.4 — explicitly scoped out per stakeholder direction).
- The `"maxpriority"` parameter referenced in the `target_threshold` field description — the RegMan's top-level `"parameters"` array is empty (`[]`), so this parameter is not itself defined anywhere in the file (see §1.4, Appendix A).

### 1.3 References

| Ref | Document | Notes |
|---|---|---|
| R1 | `INCORE_PLIC.regman` (rtl_1p0_release_regman, schema/tool version `15.4.0`) | Sole source for this VPlan |
| R2 | UVM 1.2 Register Layer (RAL) User Guide | Verification methodology reference (external, not IP-specific) |
| R3 | AMBA APB4 Protocol Specification | Bus protocol reference implied by `protocol: "APB4"` in RegMan (external standard, not read as part of this task) |

> **Information Required:** The RegMan top-level `"description"` field for INCORE_PLIC is empty, as is the `address_map` level `"description"`. No block-level functional description, application note, or architectural reference (e.g., RISC-V PLIC specification alignment) is available beyond what is stated per-register.

### 1.4 Clarifications Applied (per stakeholder confirmation)

The following interpretations were explicitly confirmed with the requester prior to drafting this VPlan, because the RegMan does not itself resolve them:

| Item | RegMan Evidence | Interpretation Applied in this VPlan |
|---|---|---|
| `"array_size": 1` on all 5 registers, vs. descriptions implying per-source/per-target multiplicity ("the respective interrupt source", "the respective target") and a 64 MB (`0x4000000`) address-map window | No parameter, array, or instance-count attribute is declared anywhere in the file to indicate more than one instance | **Flagged as Information Required.** This VPlan documents and verifies exactly **one** instance of each register at its declared offset, exactly as specified. It does **not** infer or generate additional per-source/per-target register instances, arrays, or address strides. The true source/target count must be confirmed against the architecture specification / RTL before this VPlan is scaled to a multi-instance regmodel. |
| `hw_access: "Default"` on `source_priority`, `target_enables`, `target_threshold`, `target_claim` fields (vs. explicit `"RO"` on `source_pending`) | No enumerated definition of `"Default"` is given in the file | **Flagged as Information Required.** The actual hardware-side update behavior implied by `"Default"` is undetermined. For the purpose of drafting concrete frontdoor/backdoor/mirror test cases in this VPlan, a **placeholder baseline** of "no autonomous hardware update — value changes only via declared software write" is used so that mirror-based tests (§4.1–§4.3, §6) can be authored. This placeholder is **not** a validated assumption; it must be reconfirmed against RTL/architecture documentation before test implementation sign-off (see Appendix A). |
| `target_claim` register: `sw_access: "RW"`, `read_behavior: "R"`, `write_behavior: "W"` — a plain register access declaration, despite the register description mentioning "interrupt claim/completion information" | No distinct claim-on-read / complete-on-write side-effect attribute is declared for this field | Per explicit stakeholder direction, this VPlan verifies `target_claim` **strictly as a generic RW register** per its declared `sw_access`/`read_behavior`/`write_behavior`. No claim/complete side-effect semantics are modeled, tested, or flagged as a gap in this VPlan. |
| `target_threshold` field description references a parameter `"maxpriority"` controlling its reset value, but the top-level `"parameters"` array is `[]` | `description`: *"...controlled by the parameter \"maxpriority\""*, yet `"parameters": []` at the document root | **Flagged as Information Required.** The `maxpriority` parameter is not defined anywhere in the RegMan (name, width, legal range, or default). The declared literal reset value `0b111` (`0x7`) is used as the verified reset value in this VPlan; the parameterization behavior itself is out of scope (see Appendix A). |
| `non_secure: false` / `privileged: false` on all 5 registers | Booleans present, but no behavioral/fault definition for a disallowed access is declared | **No dedicated security/privilege-mode verification** is included. Attribute values are recorded in the register summary only (§2.1) and are **not** exercised as pass/fail criteria. |

These interpretations should be re-confirmed against the RTL/architecture specification before sign-off, as they materially affect test predication and regmodel scope.

---

## 2. DUT Register Overview

### 2.1 Register Summary

Bus Interface: `APB4_interface0` — Protocol: **APB4**, Address Width: **32 bits**, Data Width: **32 bits**, Error Response: **enabled** (`error_response: true`)
Address Map: `plic0_regfile_mmap` — Address Unit: **8 bits (byte-addressed)**, Data Width: **32 bits**, Map Size: **0x4000000 (64 MB)**

| # | Register | Offset | Width (bits) | Array Size | SW Access | Reset Value | Description (verbatim from RegMan) |
|---|---|---|---|---|---|---|---|
| 1 | `source_priority` | `0x0` | 32 | 1 | RW | `0x00000000` | Regsiter holds the priority value of the respective interrupt source. *(sic — verbatim from RegMan)* |
| 2 | `source_pending` | `0x1000` | 32 | 1 | RO | `0x00000000` | Register holds the pending interrupt bits for upto 32 sources in a single register |
| 3 | `target_enables` | `0x2000` | 32 | 1 | RW | `0x00000000` | Register holds the interrupt enable bits for upto 32 sources in a single register |
| 4 | `target_threshold` | `0x200000` | 32 | 1 | RW | `0x00000007` | Register holds the priority threshold of the respective target |
| 5 | `target_claim` | `0x200004` | 32 | 1 | RW | `0x00000000` | Register holds interrupt claim/completion information for the respective target |

Common per-register attributes (identical across all 5 registers, per RegMan):
`non_secure = false`, `privileged = false`, `memory = false`, `visible_we = false`, `parity_check = false`, `presence_condition = "" (unconditionally present)`, `user_defined_properties = {}`.

> **Information Required (per §1.4):** All 5 registers declare `array_size = 1`. The 64 MB (`0x4000000`) address-map window and the "respective interrupt source" / "respective target" wording in the descriptions suggest the real DUT may support multiple interrupt sources and/or multiple targets, but no such count, parameter, or array attribute is present in the RegMan. This VPlan verifies exactly one instance of each register, as literally declared, and does not infer additional instances.

### 2.2 Register Map (Address Layout)

```
Offset          Register            Width    Notes
0x000000        source_priority     32-bit   Base of map
0x000004–0x000FFF (unmapped)                 Information Required — access behavior undefined
0x001000        source_pending      32-bit
0x001004–0x001FFF (unmapped)                 Information Required — access behavior undefined
0x002000        target_enables      32-bit
0x002004–0x1FFFFF (unmapped)                 Information Required — access behavior undefined
0x200000        target_threshold    32-bit
0x200004        target_claim        32-bit
0x200008–0x3FFFFFF (unmapped)                Information Required — access behavior undefined
                                             Map size = 0x4000000 (end of window)
```

### 2.3 Field Summary

| Register | Field | Bit Range | Reset | SW Access | HW Access | Read Behavior | Write Behavior | Description |
|---|---|---|---|---|---|---|---|---|
| `source_priority` | `source_priority` | [31:0] | `0b0` | RW | Default (Information Required, §1.4) | R | W | *(field-level description empty in RegMan; register-level description applies — see §2.1)* |
| `source_pending` | `source_pending` | [31:0] | `0b0` | RO | RO | R | NA | *(field-level description empty in RegMan; register-level description applies — see §2.1)* |
| `target_enables` | `target_enables` | [31:0] | `0b0` | RW | Default (Information Required, §1.4) | R | W | *(field-level description empty in RegMan; register-level description applies — see §2.1)* |
| `target_threshold` | `target_threshold` | [31:0] | `0b111` | RW | Default (Information Required, §1.4) | R | W | Register holds the priority threshold of respective target (IP vendors default value is '0', we changed it to actual value of '7' which is actual value observed in simulation - controlled by the parameter "maxpriority") |
| `target_claim` | `target_claim` | [31:0] | `0b0` | RW | Default (Information Required, §1.4) | R | W | *(field-level description empty in RegMan; register-level description applies — see §2.1)* |

All fields: `external_reset_source = ""` (no distinct external reset source defined — default/single reset domain applies), `write_enable = ""` (no conditional write-enable gating declared), `unlock_key = ""` (no write-lock/unlock mechanism declared), `presence_condition = ""` (unconditionally present), `enumerated_values = []` (no enumerated legal values declared for any field), `user_defined_properties = {}`.

Each register in this RegMan consists of exactly one bitfield spanning the full 32-bit register width — there are no multi-field (sub-word) registers or reserved-bit fields declared.

---

## 3. Verification Objectives

Based solely on the RegMan content, the verification of INCORE_PLIC registers shall demonstrate that:

1. **OBJ-1**: Every declared register (`source_priority`, `source_pending`, `target_enables`, `target_threshold`, `target_claim`) is correctly accessible at its declared offset via the APB4 frontdoor interface, with the declared 32-bit data width.
2. **OBJ-2**: Every field behaves per its declared `sw_access` / `read_behavior` / `write_behavior` (RW for `source_priority`, `target_enables`, `target_threshold`, `target_claim`; RO for `source_pending`).
3. **OBJ-3**: All registers/fields reset to their declared reset value following a DUT reset: `0x00000000` for `source_priority`, `source_pending`, `target_enables`, `target_claim`; `0x00000007` for `target_threshold`.
4. **OBJ-4**: The RAL model's mirror value tracks the DUT register value correctly across frontdoor writes, reads, and backdoor peek/poke operations (mirror/desired/actual consistency).
5. **OBJ-5**: `source_pending` is verified as RO — software writes of any value do not alter its state, consistent with declared `sw_access: RO` / `write_behavior: NA` / `hw_access: RO`.
6. **OBJ-6**: Accesses that violate the declared access policy (e.g., writes to `source_pending`) do not alter DUT state, consistent with declared `sw_access`.
7. **OBJ-7 (Information Required)**: The true number of interrupt sources and targets implemented by the DUT, and whether `source_priority`, `source_pending`, `target_enables`, `target_threshold`, `target_claim` are each replicated per source/target in the actual hardware — pending clarification per §1.4/§2.1.
8. **OBJ-8 (Information Required)**: The actual hardware-side update behavior implied by `hw_access: "Default"` for the four RW registers — pending clarification per §1.4.
9. **OBJ-9 (Information Required)**: Access to unmapped offsets within the `0x4000000` address window produces a well-defined response — pending clarification per §2.1/§4.5.

---

## 4. Verification Strategy

### 4.1 Frontdoor Verification Strategy

- All 5 registers shall be accessed via the UVM RAL frontdoor path over the `APB4_interface0` bus (APB4 protocol, 32-bit address, 32-bit data), exercising the RAL-generated `reg_block` for INCORE_PLIC.
- Each register/field shall be written (where `sw_access` permits) and read back through the RAL `write()` / `read()` sequence API, and results checked against the RAL mirror.
- `source_pending` shall be exercised via frontdoor **read only**, consistent with its declared `RO` access; no frontdoor write test is planned to alter its expected value (see §4.4, §4.5 for negative-write verification).
- Frontdoor tests shall use the declared 32-bit access width for all 5 registers.

### 4.2 Backdoor Verification Strategy

- Each register shall be verified via RAL `peek()` / `poke()` where a backdoor HDL path is available, to confirm the RAL model's backdoor access is consistent with the frontdoor-visible value.
- For `source_pending` (`RO`/`hw_access: RO`), backdoor `poke()` is used to inject a pending-bit pattern and confirm it is observable via a subsequent frontdoor `read()`, since this field cannot be set via a frontdoor software write.
- For the four `RW` registers with `hw_access: "Default"`, backdoor peek/poke consistency checks are planned using the placeholder baseline from §1.4 (no autonomous hardware update assumed). **Information Required:** if RTL/architecture review reveals that hardware can autonomously update these fields, the backdoor test methodology (and expected poke/read-back behavior) will need revision.

### 4.3 Register Reset Verification

- All registers/fields reset per the RegMan `reset` attribute. Reset verification shall confirm:
  - `source_priority` == `0x00000000` after reset.
  - `source_pending` == `0x00000000` after reset.
  - `target_enables` == `0x00000000` after reset.
  - `target_threshold` == `0x00000007` after reset (per declared `reset: "0b111"`; see §1.4 regarding the referenced but undefined `maxpriority` parameter).
  - `target_claim` == `0x00000000` after reset.
- No distinct `external_reset_source` is declared for any field (all are empty strings) — reset verification shall use the single/default reset domain only. No multi-domain or asynchronous-external-reset test is planned, as none is declared.

### 4.4 Register Access Policy Verification

- Verify declared `sw_access` policies exactly as specified:
  - `source_priority` — **RW**: writes update the field; reads return the last written value.
  - `source_pending` — **RO**: writes of any value are ignored/ineffective; reads return the current field value (register `sw_access: RO`, field `hw_access: RO`, `write_behavior: NA`).
  - `target_enables` — **RW**: writes update the field; reads return the last written value.
  - `target_threshold` — **RW**: writes update the field; reads return the last written value.
  - `target_claim` — **RW**: writes update the field; reads return the last written value. Per §1.4, this is verified strictly as generic RW; no claim/complete side-effect semantics are tested.
- Register-level `sw_access` for all 5 registers is consistent with their single constituent field's `sw_access`.
- Per §1.4, `non_secure`/`privileged` attribute values (`false` for all registers) are recorded for completeness (§2.1) but are **not** exercised as distinct access-policy test vectors, since the RegMan defines no fault/violation behavior to check against.

### 4.5 Negative Testing Strategy

- Frontdoor writes to `source_pending` (declared `RO`) shall be attempted with all-ones and randomized patterns; the field must not change from its current/backdoor-set value as a result of the software write attempt.
- Byte-enable / partial-access negative checks are **Information Required** — RegMan does not declare byte-strobe or sub-word access restrictions for any register (each register is a single full-width field, so partial writes are not separately defined).
- Out-of-range / unmapped-offset access testing (e.g., offsets `0x4`, `0x1800`, `0x100000`, `0x300000`) is **Information Required** pending clarification of expected behavior for the unmapped regions identified in §2.1/§2.2. `error_response: true` at the bus-interface level suggests the APB4 interconnect supports error responses, but the RegMan does not state whether every unmapped offset within the 64 MB window triggers this response.
- Multi-source/multi-target negative testing (e.g., verifying isolation between different interrupt sources' priority registers) is **Information Required / not applicable within this RegMan's scope**, since only one instance of each register is declared (§1.4).

---

## 5. Verification Features

Feature IDs are of the form `FT-<REGISTER>-<NNN>`.

### 5.1 Register: `source_priority` (offset `0x0`, 32-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-SRCPRI-001 | Frontdoor write/read of full 32-bit register | RAL frontdoor `write()`/`read()` via APB4 | Read-back value equals last written 32-bit value |
| FT-SRCPRI-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000000` |
| FT-SRCPRI-003 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read; peek matches last frontdoor-written value |
| FT-SRCPRI-004 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value; no `UVM_ERROR` on compare |
| FT-SRCPRI-005 | Boundary value test | Write `0x0`, `0xFFFFFFFF`, walking-1/walking-0 patterns | All patterns read back exactly as written (full 32-bit RW, no masked bits declared) |
| FT-SRCPRI-006 (Information Required) | Per-source instance isolation (multiple interrupt sources) | Not determinable from RegMan | Only one instance is declared (`array_size: 1`); pending clarification per §1.4/§2.1 |

### 5.2 Register: `source_pending` (offset `0x1000`, 32-bit, RO)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-SRCPEND-001 | Frontdoor read of full 32-bit register | RAL frontdoor `read()` via APB4 | Read returns current field value |
| FT-SRCPEND-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000000` |
| FT-SRCPEND-003 | RO negative-write test | RAL frontdoor `write()` with all-ones/randomized pattern, then `read()` | Register value unchanged by the write attempt (no effect, per `sw_access: RO`, `write_behavior: NA`) |
| FT-SRCPEND-004 | Backdoor poke / frontdoor read consistency | RAL `poke()` of a pending-bit pattern, then frontdoor `read()` | Frontdoor read reflects the poked value (only path to set this field, since it is RO from software) |
| FT-SRCPEND-005 | Mirror/desired/actual consistency | RAL `mirror()` after backdoor poke | RAL mirror matches DUT actual value |
| FT-SRCPEND-006 (Information Required) | Hardware-driven pending-bit set behavior (i.e., how/when hardware actually asserts pending bits) | Not determinable from RegMan | No functional description of pending-bit hardware generation is present; field is declared `hw_access: RO` only |

### 5.3 Register: `target_enables` (offset `0x2000`, 32-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-TGTEN-001 | Frontdoor write/read of full 32-bit register | RAL frontdoor `write()`/`read()` via APB4 | Read-back value equals last written 32-bit value |
| FT-TGTEN-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000000` |
| FT-TGTEN-003 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read; peek matches last written value |
| FT-TGTEN-004 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-TGTEN-005 | Boundary value test | Write `0x0`, `0xFFFFFFFF`, walking-1/walking-0 patterns | All patterns read back exactly as written |
| FT-TGTEN-006 (Information Required) | Per-source enable-bit-to-source mapping (which bit enables which of "up to 32 sources") | Not determinable beyond register-level description | Description states "up to 32 sources in a single register" but does not map specific bit positions to specific source IDs |

### 5.4 Register: `target_threshold` (offset `0x200000`, 32-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-TGTTHR-001 | Frontdoor write/read of full 32-bit register | RAL frontdoor `write()`/`read()` via APB4 | Read-back value equals last written 32-bit value |
| FT-TGTTHR-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000007` (per declared `reset: "0b111"`) |
| FT-TGTTHR-003 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read; peek matches last written value |
| FT-TGTTHR-004 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-TGTTHR-005 | Boundary value test | Write `0x0`, `0xFFFFFFFF`, walking-1/walking-0 patterns | All patterns read back exactly as written |
| FT-TGTTHR-006 (Information Required) | `maxpriority` parameter effect on reset value | Not determinable from RegMan | Description references a parameter `"maxpriority"` controlling the reset value, but no such parameter is declared in the top-level `"parameters": []` array (see §1.4, Appendix A) |
| FT-TGTTHR-007 (Information Required) | Per-target instance isolation (multiple interrupt targets) | Not determinable from RegMan | Only one instance is declared (`array_size: 1`); pending clarification per §1.4/§2.1 |

### 5.5 Register: `target_claim` (offset `0x200004`, 32-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-TGTCLAIM-001 | Frontdoor write/read of full 32-bit register | RAL frontdoor `write()`/`read()` via APB4 | Read-back value equals last written 32-bit value (verified as generic RW per §1.4) |
| FT-TGTCLAIM-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000000` |
| FT-TGTCLAIM-003 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read; peek matches last written value |
| FT-TGTCLAIM-004 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-TGTCLAIM-005 | Boundary value test | Write `0x0`, `0xFFFFFFFF`, walking-1/walking-0 patterns | All patterns read back exactly as written |

### 5.6 Address Map Features

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-MAP-001 | Correct address decode for all 5 registers | Frontdoor access at each declared offset (`0x0`, `0x1000`, `0x2000`, `0x200000`, `0x200004`) | Access reaches the intended register only; no aliasing to adjacent/unmapped space |
| FT-MAP-002 (Information Required) | Unmapped-offset access behavior | Frontdoor access to offsets outside the 5 declared registers, within the `0x4000000` window | RegMan does not define expected response (error vs. default read); pending clarification |
| FT-MAP-003 | Bus error-response capability | Negative-path frontdoor access expected to trigger bus-level error | `error_response: true` declared at bus-interface level — exact triggering conditions beyond unmapped access are not specified; scope of this feature limited to what RegMan declares |

---

## 6. Test Plan

| Test ID | Test Name | Applicable Register(s) | Category | Description |
|---|---|---|---|---|
| TC-001 | `source_priority_frontdoor_rw` | source_priority | Frontdoor R/W | Write/read source_priority via APB4 frontdoor; verify mirror match |
| TC-002 | `source_pending_frontdoor_ro_read` | source_pending | Frontdoor R/W | Frontdoor read of source_pending; verify value against RAL mirror after backdoor set |
| TC-003 | `target_enables_frontdoor_rw` | target_enables | Frontdoor R/W | Write/read target_enables via APB4 frontdoor |
| TC-004 | `target_threshold_frontdoor_rw` | target_threshold | Frontdoor R/W | Write/read target_threshold via APB4 frontdoor |
| TC-005 | `target_claim_frontdoor_rw` | target_claim | Frontdoor R/W | Write/read target_claim via APB4 frontdoor (generic RW per §1.4) |
| TC-006 | `source_priority_backdoor_peek_poke` | source_priority | Backdoor | RAL `peek()`/`poke()`; cross-check with frontdoor |
| TC-007 | `source_pending_backdoor_peek_poke` | source_pending | Backdoor | RAL `poke()` of pending-bit pattern; verify via frontdoor `read()` and `peek()` |
| TC-008 | `target_enables_backdoor_peek_poke` | target_enables | Backdoor | RAL `peek()`/`poke()`; cross-check with frontdoor |
| TC-009 | `target_threshold_backdoor_peek_poke` | target_threshold | Backdoor | RAL `peek()`/`poke()`; cross-check with frontdoor |
| TC-010 | `target_claim_backdoor_peek_poke` | target_claim | Backdoor | RAL `peek()`/`poke()`; cross-check with frontdoor |
| TC-011 | `all_regs_reset_value_check` | source_priority, source_pending, target_enables, target_threshold, target_claim | Reset | Apply reset; verify each register equals its declared reset value (`0x0` for four registers, `0x7` for target_threshold) |
| TC-012 | `source_pending_ro_negative_write` | source_pending | Access Permission (RO) | Attempt frontdoor write with all-ones/randomized pattern; verify no change to register value |
| TC-013 | `source_priority_field_rw_check` | source_priority | Access Permission (RW) | Verify field is fully RW per declared access |
| TC-014 | `target_enables_field_rw_check` | target_enables | Access Permission (RW) | Verify field is fully RW per declared access |
| TC-015 | `target_threshold_field_rw_check` | target_threshold | Access Permission (RW) | Verify field is fully RW per declared access |
| TC-016 | `target_claim_field_rw_check` | target_claim | Access Permission (RW) | Verify field is fully RW per declared access |
| TC-017 | `all_regs_mirror_update_check` | source_priority, source_pending, target_enables, target_threshold, target_claim | Mirror/Update | RAL `mirror()`/`update()` calls after writes/pokes; confirm no mismatch reported |
| TC-018 | `ral_built_in_hdl_reg_access_seq` | source_priority, source_pending, target_enables, target_threshold, target_claim | Random Access | Standard UVM RAL built-in sequences (`uvm_reg_hw_reset_seq`, `uvm_reg_bit_bash_seq`, `uvm_reg_access_seq`) applied to all registers |
| TC-019 | `rw_regs_boundary_values` | source_priority, target_enables, target_threshold, target_claim | Boundary | Write `0x0`, `0xFFFFFFFF`, walking-1/walking-0 patterns to each RW register; verify exact read-back |
| TC-020 (Information Required) | `unmapped_offset_access` | Address map | Negative | Access offsets outside the 5 declared registers within the `0x4000000` window — expected response pending clarification (§2.1, §4.5) |
| TC-021 (Information Required) | `multi_source_target_isolation` | source_priority, target_enables, target_threshold, target_claim | Negative / Coverage | Verify isolation across multiple interrupt sources/targets — not testable since only 1 instance of each register is declared (§1.4) |

---

## 7. Functional Coverage Plan

| Coverage Group | Coverpoints | Source |
|---|---|---|
| `cg_register_access` | Per-register: read hit, write hit (write hit N/A for `source_pending`), for all 5 registers | Register list, §2.1 |
| `cg_source_priority_values` | Value bins: `0x0`, all-ones, min/max boundary, random mid-range | Boundary test plan, §6 TC-019 |
| `cg_source_pending_values` | Value bins as set via backdoor poke: `0x0`, all-ones, single-bit-set patterns | §4.2, §6 TC-002/TC-007 |
| `cg_target_enables_values` | Value bins: `0x0`, all-ones, min/max boundary, random mid-range | Boundary test plan, §6 TC-019 |
| `cg_target_threshold_values` | Value bins: reset value (`0x7`), `0x0`, all-ones, min/max boundary, random mid-range | Boundary test plan, §6 TC-019 |
| `cg_target_claim_values` | Value bins: `0x0`, all-ones, min/max boundary, random mid-range | Boundary test plan, §6 TC-019 |
| `cg_access_type` | Frontdoor read, frontdoor write (where applicable), backdoor peek, backdoor poke — crossed with each register | §4.1, §4.2 |
| `cg_reset_coverage` | Reset applied + register read = expected reset value, per register | §4.3 |
| `cg_access_policy` | RO-register write-attempt (`source_pending`, no effect) vs. RW-register write (effective, other 4 registers) | §4.4 |
| `cg_unmapped_access` (Information Required) | Access-outside-map coverpoint — bin definition depends on clarified expected behavior (§2.1) | Pending |
| `cg_multi_instance` (Information Required) | Per-source / per-target coverage crosses — not definable since only 1 instance per register is declared | Pending, §1.4 |

> No RegMan-declared `enumerated_values` exist for any field (all are `[]`), so no enumerated-value coverage bins are defined; only reset/boundary/random value bins are applicable.

---

## 8. Requirements Traceability Matrix (RTM)

| Req ID | Requirement | Register/Field | Planned Test(s) | Verification Method |
|---|---|---|---|---|
| REQ-01 | `source_priority` accessible at offset 0x0, 32-bit RW | source_priority | TC-001, TC-011, TC-013 | Frontdoor, Reset, Access Policy |
| REQ-02 | `source_priority` resets to 0x00000000 | source_priority | TC-011 | Reset |
| REQ-03 | `source_priority` full-width boundary values are read/write consistent | source_priority | TC-019 | Boundary |
| REQ-04 | `source_pending` accessible at offset 0x1000, 32-bit RO | source_pending | TC-002, TC-011 | Frontdoor, Reset |
| REQ-05 | `source_pending` resets to 0x00000000 | source_pending | TC-011 | Reset |
| REQ-06 | `source_pending` rejects software writes (no state change) | source_pending | TC-012 | Access Policy — Negative |
| REQ-07 | `source_pending` value observable via backdoor poke + frontdoor read | source_pending | TC-007 | Backdoor |
| REQ-08 | `target_enables` accessible at offset 0x2000, 32-bit RW | target_enables | TC-003, TC-011, TC-014 | Frontdoor, Reset, Access Policy |
| REQ-09 | `target_enables` resets to 0x00000000 | target_enables | TC-011 | Reset |
| REQ-10 | `target_enables` full-width boundary values are read/write consistent | target_enables | TC-019 | Boundary |
| REQ-11 | `target_threshold` accessible at offset 0x200000, 32-bit RW | target_threshold | TC-004, TC-011, TC-015 | Frontdoor, Reset, Access Policy |
| REQ-12 | `target_threshold` resets to 0x00000007 | target_threshold | TC-011 | Reset |
| REQ-13 | `target_threshold` full-width boundary values are read/write consistent | target_threshold | TC-019 | Boundary |
| REQ-14 | `target_claim` accessible at offset 0x200004, 32-bit RW | target_claim | TC-005, TC-011, TC-016 | Frontdoor, Reset, Access Policy |
| REQ-15 | `target_claim` resets to 0x00000000 | target_claim | TC-011 | Reset |
| REQ-16 | RAL mirror tracks DUT state across all registers | all 5 registers | TC-017, TC-018 | Mirror/Update |
| REQ-17 | Backdoor peek/poke consistent with frontdoor state | all 5 registers | TC-006–TC-010 | Backdoor |
| REQ-18 (Information Required) | Unmapped address-space access behavior defined | Address map (0x4000000 window) | TC-020 | Negative — pending clarification |
| REQ-19 (Information Required) | True interrupt source / target instance count defined | source_priority, source_pending, target_enables, target_threshold, target_claim | TC-021 | Not testable from RegMan alone — pending architecture spec |
| REQ-20 (Information Required) | `maxpriority` parameter defined and its effect on `target_threshold` reset value confirmed | target_threshold | FT-TGTTHR-006 | Not testable from RegMan alone — parameter not declared |
| REQ-21 (Information Required) | Actual hardware-update behavior for `hw_access: "Default"` fields confirmed | source_priority, target_enables, target_threshold, target_claim | FT-SRCPRI-*, FT-TGTEN-*, FT-TGTTHR-*, FT-TGTCLAIM-* (backdoor/mirror subset) | Not testable from RegMan alone — pending architecture spec |

---

## 9. Regression Plan

- **Regression Scope**: All tests in §6 (TC-001 through TC-019) form the baseline regression suite for every code/RTL change affecting INCORE_PLIC, using the RAL model generated from `INCORE_PLIC.regman` (rtl_1p0_release_regman variant).
- **Regression Composition**:
  - Directed tests: TC-001–TC-016 (deterministic per-register/per-field checks).
  - Built-in RAL sequences (random/generic): TC-018, run with multiple random seeds per regression cycle.
  - Boundary tests: TC-019, run with fixed corner-case values every regression, plus randomized mid-range values per seed.
- **Pass/Fail Criteria**: No RAL `UVM_ERROR`/`UVM_FATAL` on register mirror mismatch; all declared reset values, access policies, and boundary values match exactly.
- **Exclusions Pending Clarification**: TC-020, TC-021 are **not** included in the baseline regression pass/fail gate until the "Information Required" items in §1.4, §2.1, §4.5 are resolved with the register/architecture owner. They are tracked as open items and should be added once clarified.
- **Seed/Iteration Policy**: Information Required — the RegMan does not define a required regression seed count, iteration count, or coverage closure threshold; this is a testbench/methodology decision outside the scope of the RegMan and should be defined by the verification team.
- **Re-run Triggers**: Any change to `INCORE_PLIC.regman` (offsets, widths, access types, reset values) should trigger regeneration of the RAL model and a full re-run of this regression suite.

---

## Appendix A: Open Items Requiring Clarification (Consolidated)

| # | Item | Section | Impact |
|---|---|---|---|
| 1 | True number of interrupt sources and targets implemented by the DUT (RegMan declares `array_size: 1` for all registers despite "respective source/target" wording and a 64 MB address window) | §1.4, §2.1, TC-021, REQ-19 | Fundamental scope of the RAL regmodel and per-instance test/coverage strategy |
| 2 | Actual hardware-side update behavior implied by `hw_access: "Default"` on `source_priority`, `target_enables`, `target_threshold`, `target_claim` | §1.4, §4.2, REQ-21 | Whether mirror-based frontdoor/backdoor test methodology (assumed as placeholder) remains valid |
| 3 | `maxpriority` parameter referenced in `target_threshold`'s description but absent from the top-level `parameters: []` array | §1.4, §2.3, FT-TGTTHR-006, REQ-20 | Confirmation of the `target_threshold` reset-value rationale; parameterized-reset test coverage |
| 4 | Expected behavior for accesses to unmapped offsets within the `0x4000000` address window | §2.1, §4.5, TC-020, REQ-18 | Negative test pass/fail criteria, functional coverage bins |
| 5 | Interrupt arbitration/priority-resolution behavior across `source_priority`, `target_threshold`, and `target_claim` | §1.2 (out of scope) | Whether cross-register functional tests are needed beyond independent register access |
| 6 | Byte-enable / partial (sub-word) access restrictions, if any | §4.5 | Whether partial-write negative tests are required |
| 7 | Regression seed count / coverage closure threshold | §9 | Regression sizing and exit criteria |

This VPlan should be reviewed against the RTL/UVM regmodel and the architecture specification to resolve the items above prior to test implementation sign-off.
