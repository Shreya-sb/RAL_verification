# Verification Plan: GIC600 Register Verification (UVM RAL)

| | |
|---|---|
| **Document Title** | Verification Plan – GIC600 (gic600_1) Register Verification |
| **DUT / IP** | GIC600 (instance name `gic600_1`) |
| **IP Version** | `r1p6_0` |
| **Vendor** | arm.com |
| **Library** | CoreLink |
| **Role** | peripheral |
| **Source of Truth** | `GIC600.regman` (rtl_1p0_release_regman) |
| **Verification Method** | UVM Register Abstraction Layer (RAL) |
| **Document Status** | Draft – Generated strictly from RegMan content |
| **Date** | 2026-07-20 |

---

## 1. Document Overview

### 1.1 Purpose

This document defines the Verification Plan (VPlan) for the register space of the **GIC600** (`gic600_1`) peripheral, to be verified using the UVM Register Abstraction Layer (RAL). The plan defines verification objectives, strategy, planned features, test plan, functional coverage, and traceability for all registers and fields exposed via the `ACELite_Slave` bus interface, exactly as declared in the RegMan source file.

### 1.2 Scope

This VPlan covers **only** the register content explicitly defined in the file:

```
/data/work_area/shreya_sb/claude_code/Regression_26/Rjnsoc/system_reg_map/rjn_soc_reg_map/rtl_1p0_release_regman/GIC600.regman
```

No other files (RTL, testbench, UVM regmodel `.sv`, `.rdl`, `.vc`, VHDL sources, GICv3/GIC-600 Architecture Specification/TRM, or prior documentation) were read, referenced, or used to infer any information in this document, per verification-plan authoring instructions. Any behavior, timing, hardware interaction, or configuration detail not explicitly present in the RegMan is marked **"Information Required"** rather than assumed.

**In scope:**
- All **27 registers** and their **626 bitfields**, declared under bus interface `ACELite_Slave` / address map `ACELite_Slave_MM` (size `0x200`, byte-addressed, 64-bit data path per `address_unit_bits: 8`, `data_width: 64`):
  `GICD_CTLR`, `GICD_TYPER`, `GICD_IIDR`, `GICD_FCTLR`, `GICD_SAC`, `GICD_SETSPI_NSR`, `GICD_CLRSPI_NSR`, `GICD_SETSPI_SR`, `GICD_CLRSPI_SR`, `GICD_IGROUPR1`–`GICD_IGROUPR6`, `GICD_ISENABLER1`–`GICD_ISENABLER6`, `GICD_ICENABLER1`–`GICD_ICENABLER6`.
- Frontdoor (`ACELite_Slave`) and backdoor (RAL `peek()`/`poke()`) register access verification.
- Reset value, access policy (`RO`/`RW`/`WO`), and negative-access verification based solely on RegMan-declared attributes.

**Out of scope / not determinable from the RegMan:**
- The 34 top-level `"parameters"` entries (`SPI_INV`, `SPI_SYNC`, and 32 per-PPI `GIC600_1_PPI0_0_<n>_SYNC`/`_INV`, `n = 16..31`). These are RTL elaboration-time parameters (`"rtl_parameter": true`), not software-visible registers in the `ACELite_Slave` address map, and are **not** part of the RAL regmodel or this VPlan's register verification scope.
- Any GIC600/GICv3 register **not explicitly declared** in this RegMan — e.g. `GICD_IGROUPR0`, `GICD_ISENABLER0`, `GICD_ICENABLER0` (SGI/PPI banked registers), `GICD_ISPENDR*`, `GICD_ICPENDR*`, `GICD_ISACTIVER*`, `GICD_ICACTIVER*`, `GICD_IPRIORITYR*`, `GICD_ITARGETSR*`/`GICD_IROUTER*`, `GICD_ICFGR*`, `GICD_NSACR*`, `GICD_SGIR`, `GICD_CPENDSGIR*`, `GICD_SPENDSGIR*`, Redistributor registers, or ITS registers. None of these appear anywhere in the file; this VPlan does not invent them and treats their absence as **out of scope**, not as a gap to be filled.
- Any second bus interface, address map, or memory-mapped block — only **one** `bus_interfaces` entry (`ACELite_Slave`) and **one** `address_map` (`ACELite_Slave_MM`) are declared.
- The true hardware-side behavior implied by `hw_access: "Default"` (626 of 626 fields use either `"RO"` or `"Default"` — no field uses any other value, and `"Default"` itself is never enumerated/defined anywhere in the file).
- Interrupt-signal-level functional behavior (SPI set/clear propagation to interrupt lines, group/enable interaction with the CPU interface, wake/power signaling, etc.) — the RegMan declares only static per-bit register attributes, not interrupt-controller functional behavior.

### 1.3 References

| Ref | Document | Notes |
|---|---|---|
| R1 | `GIC600.regman` (rtl_1p0_release_regman, schema/tool version `15.4.0`) | Sole source for this VPlan |
| R2 | UVM 1.2 Register Layer (RAL) User Guide | Verification methodology reference (external, not IP-specific) |
| R3 | AMBA ACE-Lite Protocol Specification | Bus protocol reference implied by the `ACELite_Slave` interface name and description (external standard, not read as part of this task) |

> **Information Required:** The RegMan top-level `"description"` for `gic600_1` is `"Structural stiched file"` (verbatim, including spelling) — no functional/architectural description of the Distributor block, nor a reference to a specific GICv3/GIC-600 architecture document revision, is present in the file.

### 1.4 Clarifications Applied (per stakeholder confirmation)

The following interpretations were explicitly confirmed with the requester prior to drafting this VPlan, because the RegMan does not itself resolve them:

| Item | RegMan Evidence | Interpretation Applied in this VPlan |
|---|---|---|
| 18 of the 27 registers (`GICD_IGROUPR1`–`6`, `GICD_ISENABLER1`–`6`, `GICD_ICENABLER1`–`6`) each declare 32 identical single-bit fields (576 of 626 total fields), following an exact repeating naming/attribute pattern per register (`group_status_bit<31..0>`, `set_enable_bit<31..0>`, `clear_enable_bit<31..0>`) | Verified programmatically: all 576 fields share identical `width=1`, `reset="0b0"`, `sw_access="RW"`, `hw_access="Default"`, `read_behavior="R"`, `write_behavior="W"`, `enumerated_values=[]` | **Confirmed:** Sections 2.3, 5, and 6 represent these using **consolidated pattern entries** (one feature/test description per register documenting the repeating bit group, plus a bit-bash/walking-1 test per register) rather than 576 individual rows. All 626 fields are covered functionally; none are omitted from the verification objective, only from exhaustive per-row enumeration. |
| Numerous `RESERVED*` bitfields (e.g. `RESERVED1` in `GICD_CTLR`, `RESERVED0`–`RESERVED4` in `GICD_FCTLR`, `RESERVED0` in `GICD_SAC`/`GICD_SETSPI_NSR`/`GICD_CLRSPI_NSR`/`GICD_SETSPI_SR`/`GICD_CLRSPI_SR`) are declared `sw_access: "RW"` (or `"WO"` where the parent register is `WO`), `hw_access: "Default"`, `read_behavior: "R"`, `write_behavior: "W"` — i.e., literally writable and readable (or writable, for `WO` registers), **not** the conventional RO/SBZP treatment usually applied to reserved bits | No `presence_condition`, no distinct reserved-bit access override — the field-level `sw_access` is the only declared access attribute, identical in form to non-reserved fields in the same register | **Confirmed:** These bits are verified **literally as their declared access type** (RW or WO), exactly like any other field of that type — full write/read-back, reset-value, and boundary checks apply. This is flagged as an unusual-but-explicit declaration (not the conventional RO/SBZP convention) in §2.3/§5, but is **not** excluded from mandatory checks, since doing so would itself be an assumption beyond the RegMan. |
| `GICD_ICENABLER1`–`6` fields (`clear_enable_bit0`–`31`) are declared **identically** to `GICD_ISENABLER1`–`6` fields (`set_enable_bit0`–`31`): `sw_access: "RW"`, `hw_access: "Default"`, `read_behavior: "R"`, `write_behavior: "W"` — no write-1-to-clear (W1C) encoding, no distinct `read_behavior`/`write_behavior` value, and no `enumerated_values` anywhere in the file, even though GICv3 architecture conventionally gives `ICENABLER` registers write-1-to-clear semantics | No W1C-specific attribute exists in the RegMan schema instance for these fields | **Confirmed:** `GICD_ICENABLER1`–`6` are verified **strictly as literal generic RW** fields (write value, read back same value) — consistent with how the analogous ambiguity was resolved for `target_claim` in the prior INCORE_PLIC VPlan. The RegMan/architecture discrepancy (no declared W1C behavior for a register conventionally expected to have it) is recorded as an **Information Required** item (§4.4, Appendix A) for RTL/architecture confirmation before test-implementation sign-off; no W1C-specific test cases are authored in this VPlan, since doing so would be inferring behavior not present in the RegMan. |
| `hw_access: "Default"` appears on 610 of 626 fields (all fields except the 16 pure-`RO` fields in `GICD_TYPER`/`GICD_IIDR`, which use `hw_access: "RO"`); `"Default"` itself is never enumerated or defined anywhere in the file | No enumerated definition of `"Default"` is given in the file (same ambiguity pattern as the prior INCORE_PLIC VPlan) | **Flagged as Information Required**, consistent with the prior VPlan's precedent. A placeholder baseline of "no autonomous hardware update — value changes only via declared software write" is used so that mirror-based test cases (§4.1–§4.3, §6) can be authored. This placeholder is **not** a validated assumption and must be reconfirmed against RTL/architecture documentation before test-implementation sign-off (Appendix A). |

These interpretations should be re-confirmed against the RTL/UVM regmodel and the GIC-600 architecture specification before sign-off, as they materially affect test predication and regmodel scope.

---

## 2. DUT Register Overview

### 2.1 Register Summary

Bus Interface: `ACELite_Slave` — Protocol: **Other** *(declared literally as `"Other"` in RegMan; description: "ACE-Lite Slave Programming Interface")*, Address Width: **32 bits**, Data Width: **64 bits**, Error Response: **enabled** (`error_response: true`)
Address Map: `ACELite_Slave_MM` — Address Unit: **8 bits (byte-addressed)**, Data Width: **64 bits**, Map Size: **0x200 (512 bytes)**

All 27 registers share these common attributes (verified identical across every register in the RegMan): `array_size = 1`, `non_secure = false`, `privileged = false`, `memory = false`, `visible_we = "false"`, `parity_check = false`, `presence_condition = "" (unconditionally present)`, `user_defined_properties = {}`.

| # | Register | Offset | Width | SW Access | Reset Value | # Fields | Description (verbatim from RegMan) |
|---|---|---|---|---|---|---|---|
| 1 | `GICD_CTLR` | `0x0` | 32 | RW | `0x00000070` | 10 | GICD_CTLR |
| 2 | `GICD_TYPER` | `0x4` | 32 | RO | `0x00790006` | 12 | GICD_TYPER |
| 3 | `GICD_IIDR` | `0x8` | 32 | RO | `0x0201743B` | 5 | GICD_IIDR |
| 4 | `GICD_FCTLR` | `0x20` | 32 | RW | `0x00000000` | 11 | GICD_FCTLR |
| 5 | `GICD_SAC` | `0x24` | 32 | RW | `0x00000000` | 4 | GICD_SAC |
| 6 | `GICD_SETSPI_NSR` | `0x40` | 32 | WO | `0x00000000` | 2 | GICD_SETSPI_NSR |
| 7 | `GICD_CLRSPI_NSR` | `0x48` | 32 | WO | `0x00000000` | 2 | GICD_CLRSPI_NSR |
| 8 | `GICD_SETSPI_SR` | `0x50` | 32 | WO | `0x00000000` | 2 | GICD_SETSPI_SR |
| 9 | `GICD_CLRSPI_SR` | `0x58` | 32 | WO | `0x00000000` | 2 | GICD_CLRSPI_SR |
| 10 | `GICD_IGROUPR1` | `0x84` | 32 | RW | `0x00000000` | 32 | GICD_IGROUPR1 |
| 11 | `GICD_IGROUPR2` | `0x88` | 32 | RW | `0x00000000` | 32 | GICD_IGROUPR2 |
| 12 | `GICD_IGROUPR3` | `0x8C` | 32 | RW | `0x00000000` | 32 | GICD_IGROUPR3 |
| 13 | `GICD_IGROUPR4` | `0x90` | 32 | RW | `0x00000000` | 32 | GICD_IGROUPR4 |
| 14 | `GICD_IGROUPR5` | `0x94` | 32 | RW | `0x00000000` | 32 | GICD_IGROUPR5 |
| 15 | `GICD_IGROUPR6` | `0x98` | 32 | RW | `0x00000000` | 32 | GICD_IGROUPR6 |
| 16 | `GICD_ISENABLER1` | `0x104` | 32 | RW | `0x00000000` | 32 | GICD_ISENABLER1 |
| 17 | `GICD_ISENABLER2` | `0x108` | 32 | RW | `0x00000000` | 32 | GICD_ISENABLER2 |
| 18 | `GICD_ISENABLER3` | `0x10C` | 32 | RW | `0x00000000` | 32 | GICD_ISENABLER3 |
| 19 | `GICD_ISENABLER4` | `0x110` | 32 | RW | `0x00000000` | 32 | GICD_ISENABLER4 |
| 20 | `GICD_ISENABLER5` | `0x114` | 32 | RW | `0x00000000` | 32 | GICD_ISENABLER5 |
| 21 | `GICD_ISENABLER6` | `0x118` | 32 | RW | `0x00000000` | 32 | GICD_ISENABLER6 |
| 22 | `GICD_ICENABLER1` | `0x184` | 32 | RW | `0x00000000` | 32 | GICD_ICENABLER1 |
| 23 | `GICD_ICENABLER2` | `0x188` | 32 | RW | `0x00000000` | 32 | GICD_ICENABLER2 |
| 24 | `GICD_ICENABLER3` | `0x18C` | 32 | RW | `0x00000000` | 32 | GICD_ICENABLER3 |
| 25 | `GICD_ICENABLER4` | `0x190` | 32 | RW | `0x00000000` | 32 | GICD_ICENABLER4 |
| 26 | `GICD_ICENABLER5` | `0x194` | 32 | RW | `0x00000000` | 32 | GICD_ICENABLER5 |
| 27 | `GICD_ICENABLER6` | `0x198` | 32 | RW | `0x00000000` | 32 | GICD_ICENABLER6 |

Register-level `sw_access` distribution: **21 RW**, **2 RO** (`GICD_TYPER`, `GICD_IIDR`), **4 WO** (`GICD_SETSPI_NSR`, `GICD_CLRSPI_NSR`, `GICD_SETSPI_SR`, `GICD_CLRSPI_SR`). Total fields declared: **626** (across 27 registers).

> **Information Required:** No register-level `description` in the file is more descriptive than the register's own name (i.e., `"description": "GICD_CTLR"` for the `GICD_CTLR` register, etc.) — there is no prose functional description for any register beyond its name and its per-bitfield descriptions (§2.3).

### 2.2 Register Map (Address Layout)

```
Offset          Register             Width    Notes
0x000           GICD_CTLR            32-bit   Base of map
0x004           GICD_TYPER           32-bit
0x008           GICD_IIDR            32-bit
0x00C – 0x01F   (unmapped/reserved)           Information Required — not declared in RegMan
0x020           GICD_FCTLR           32-bit
0x024           GICD_SAC             32-bit
0x028 – 0x03F   (unmapped/reserved)           Information Required — not declared in RegMan
0x040           GICD_SETSPI_NSR      32-bit
0x044 – 0x047   (unmapped/reserved)           Information Required — not declared in RegMan
0x048           GICD_CLRSPI_NSR      32-bit
0x04C – 0x04F   (unmapped/reserved)           Information Required — not declared in RegMan
0x050           GICD_SETSPI_SR       32-bit
0x054 – 0x057   (unmapped/reserved)           Information Required — not declared in RegMan
0x058           GICD_CLRSPI_SR       32-bit
0x05C – 0x083   (unmapped/reserved)           Information Required — includes would-be GICD_IGROUPR0 (0x080), not declared
0x084           GICD_IGROUPR1        32-bit
0x088           GICD_IGROUPR2        32-bit
0x08C           GICD_IGROUPR3        32-bit
0x090           GICD_IGROUPR4        32-bit
0x094           GICD_IGROUPR5        32-bit
0x098           GICD_IGROUPR6        32-bit
0x09C – 0x103   (unmapped/reserved)           Information Required — includes would-be GICD_ISENABLER0 (0x100), not declared
0x104           GICD_ISENABLER1      32-bit
0x108           GICD_ISENABLER2      32-bit
0x10C           GICD_ISENABLER3      32-bit
0x110           GICD_ISENABLER4      32-bit
0x114           GICD_ISENABLER5      32-bit
0x118           GICD_ISENABLER6      32-bit
0x11C – 0x183   (unmapped/reserved)           Information Required — includes would-be GICD_ICENABLER0 (0x180), not declared
0x184           GICD_ICENABLER1      32-bit
0x188           GICD_ICENABLER2      32-bit
0x18C           GICD_ICENABLER3      32-bit
0x190           GICD_ICENABLER4      32-bit
0x194           GICD_ICENABLER5      32-bit
0x198           GICD_ICENABLER6      32-bit
0x19C – 0x1FF   (unmapped/reserved)           Information Required — end of declared map (size = 0x200)
```

> **Information Required:** The RegMan does not define expected access behavior (error response, RAZ/WI, or undefined) for any of the unmapped offset ranges above, despite `error_response: true` being declared at the bus-interface level. See §4.5 and Appendix A.
>
> The gaps at `0x080`, `0x100`, and `0x180` (4 bytes each) are exactly where `GICD_IGROUPR0`, `GICD_ISENABLER0`, and `GICD_ICENABLER0` would conventionally reside in a GICv3-family Distributor (these registers are typically banked per-PE for SGI/PPI interrupt IDs 0–31 and are not exposed via this shared-frame address map). This is consistent with `GICD_TYPER.ITLinesNumber = 0b110 = 6`, which yields `32×(ITLinesNumber+1) = 224` supported interrupt IDs (0–223), of which IDs 32–223 (192 SPIs = 6 register-words) are covered by the 6 instances each of `IGROUPR`/`ISENABLER`/`ICENABLER` declared here. This is stated as an observation for reviewer context only — it is **not** used to infer or invent any undeclared register.

### 2.3 Field Summary

#### 2.3.1 Scalar / Multi-Field Registers (9 registers, 50 fields)

| Register | Field | Bit Range | Reset | SW Access | HW Access | R/W Behavior | Description |
|---|---|---|---|---|---|---|---|
| `GICD_CTLR` | `RWP` | [31:31] | `0b0` | RO | RO | R / NA | RWP |
| `GICD_CTLR` | `RESERVED1` | [30:8] | `0b0` | RW *(literal, §1.4)* | Default | R / W | RESERVED1 |
| `GICD_CTLR` | `E1NWF` | [7:7] | `0b0` | RW | Default | R / W | E1NWF |
| `GICD_CTLR` | `DS` | [6:6] | `0b1` | RO | RO | R / NA | DS |
| `GICD_CTLR` | `ARE_NS` | [5:5] | `0b1` | RW | Default | R / W | ARE_NS |
| `GICD_CTLR` | `ARE_S` | [4:4] | `0b1` | RW | Default | R / W | ARE_S |
| `GICD_CTLR` | `RESERVED0` | [3:3] | `0b0` | RW *(literal, §1.4)* | Default | R / W | RESERVED0 |
| `GICD_CTLR` | `EnableGrp1_s` | [2:2] | `0b0` | RW | Default | R / W | EnableGrp1_s |
| `GICD_CTLR` | `EnableGrp1_ns` | [1:1] | `0b0` | RW | Default | R / W | EnableGrp1_ns |
| `GICD_CTLR` | `EnableGrp0` | [0:0] | `0b0` | RW | Default | R / W | EnableGrp0 |
| `GICD_TYPER` | `RESERVED1` | [31:26] | `0b0` | RO | RO | R / NA | RESERVED1 |
| `GICD_TYPER` | `No1N` | [25:25] | `0b0` | RO | RO | R / NA | No1N |
| `GICD_TYPER` | `A3V` | [24:24] | `0b0` | RO | RO | R / NA | A3V |
| `GICD_TYPER` | `IDbits` | [23:19] | `0b1111` | RO | RO | R / NA | IDbits |
| `GICD_TYPER` | `DVIS` | [18:18] | `0b0` | RO | RO | R / NA | DVIS |
| `GICD_TYPER` | `LPIS` | [17:17] | `0b0` | RO | RO | R / NA | LPIS |
| `GICD_TYPER` | `MBIS` | [16:16] | `0b1` | RO | RO | R / NA | MBIS |
| `GICD_TYPER` | `LSPI` | [15:11] | `0b0` | RO | RO | R / NA | LSPI |
| `GICD_TYPER` | `SecurityExtn` | [10:10] | `0b0` | RO | RO | R / NA | SecurityExtn |
| `GICD_TYPER` | `RESERVED0` | [9:8] | `0b0` | RO | RO | R / NA | RESERVED0 |
| `GICD_TYPER` | `CPUNumber` | [7:5] | `0b0` | RO | RO | R / NA | CPUNumber |
| `GICD_TYPER` | `ITLinesNumber` | [4:0] | `0b110` | RO | RO | R / NA | ITLinesNumber |
| `GICD_IIDR` | `ProductID` | [31:24] | `0b10` | RO | RO | R / NA | ProductID |
| `GICD_IIDR` | `RESERVED0` | [23:20] | `0b0` | RO | RO | R / NA | RESERVED0 |
| `GICD_IIDR` | `Variant` | [19:16] | `0b1` | RO | RO | R / NA | Variant |
| `GICD_IIDR` | `Revision` | [15:12] | `0b111` | RO | RO | R / NA | Revision |
| `GICD_IIDR` | `Implementer` | [11:0] | `0b10000111011` | RO | RO | R / NA | Implementer |
| `GICD_FCTLR` | `RESERVED4` | [31:27] | `0b0` | RW *(literal, §1.4)* | Default | R / W | RESERVED4 |
| `GICD_FCTLR` | `POS` | [26:26] | `0b0` | RW | Default | R / W | POS |
| `GICD_FCTLR` | `QDENY` | [25:25] | `0b0` | RW | Default | R / W | QDENY |
| `GICD_FCTLR` | `RESERVED3` | [24:22] | `0b0` | RW *(literal, §1.4)* | Default | R / W | RESERVED3 |
| `GICD_FCTLR` | `DCC` | [21:21] | `0b0` | RW | Default | R / W | DCC |
| `GICD_FCTLR` | `RESERVED2` | [20:18] | `0b0` | RW *(literal, §1.4)* | Default | R / W | RESERVED2 |
| `GICD_FCTLR` | `NSACR` | [17:16] | `0b0` | RW | Default | R / W | NSACR |
| `GICD_FCTLR` | `RESERVED1` | [15:13] | `0b0` | RW *(literal, §1.4)* | Default | R / W | RESERVED1 |
| `GICD_FCTLR` | `CGO` | [12:4] | `0b0` | RW | Default | R / W | CGO |
| `GICD_FCTLR` | `RESERVED0` | [3:1] | `0b0` | RW *(literal, §1.4)* | Default | R / W | RESERVED0 |
| `GICD_FCTLR` | `SIP` | [0:0] | `0b0` | RW | Default | R / W | SIP |
| `GICD_SAC` | `RESERVED0` | [31:3] | `0b0` | RW *(literal, §1.4)* | Default | R / W | RESERVED0 |
| `GICD_SAC` | `GICPNS` | [2:2] | `0b0` | RW | Default | R / W | GICPNS |
| `GICD_SAC` | `GICTNS` | [1:1] | `0b0` | RW | Default | R / W | GICTNS |
| `GICD_SAC` | `DSL` | [0:0] | `0b0` | RW | Default | R / W | DSL |
| `GICD_SETSPI_NSR` | `RESERVED0` | [31:10] | `0b0` | WO *(literal, §1.4)* | Default | NA / W | RESERVED0 |
| `GICD_SETSPI_NSR` | `ID` | [9:0] | `0b0` | WO | Default | NA / W | ID |
| `GICD_CLRSPI_NSR` | `RESERVED0` | [31:10] | `0b0` | WO *(literal, §1.4)* | Default | NA / W | RESERVED0 |
| `GICD_CLRSPI_NSR` | `ID` | [9:0] | `0b0` | WO | Default | NA / W | ID |
| `GICD_SETSPI_SR` | `RESERVED0` | [31:10] | `0b0` | WO *(literal, §1.4)* | Default | NA / W | RESERVED0 |
| `GICD_SETSPI_SR` | `ID` | [9:0] | `0b0` | WO | Default | NA / W | ID |
| `GICD_CLRSPI_SR` | `RESERVED0` | [31:10] | `0b0` | WO *(literal, §1.4)* | Default | NA / W | RESERVED0 |
| `GICD_CLRSPI_SR` | `ID` | [9:0] | `0b0` | WO | Default | NA / W | ID |

#### 2.3.2 Consolidated Bit-Array Registers (18 registers, 576 fields — per §1.4)

| Register Group | Instances (Offsets) | Bit Field Naming Pattern | Bit Range (each instance) | Reset | SW Access | HW Access | R/W Behavior |
|---|---|---|---|---|---|---|---|
| `GICD_IGROUPR1`–`6` | `0x84, 0x88, 0x8C, 0x90, 0x94, 0x98` | `group_status_bit31` … `group_status_bit0` (32 fields/register, 1 bit each) | [31:0] (32 independent 1-bit fields) | `0b0` (all bits) | RW | Default | R / W |
| `GICD_ISENABLER1`–`6` | `0x104, 0x108, 0x10C, 0x110, 0x114, 0x118` | `set_enable_bit31` … `set_enable_bit0` (32 fields/register, 1 bit each) | [31:0] (32 independent 1-bit fields) | `0b0` (all bits) | RW | Default | R / W |
| `GICD_ICENABLER1`–`6` | `0x184, 0x188, 0x18C, 0x190, 0x194, 0x198` | `clear_enable_bit31` … `clear_enable_bit0` (32 fields/register, 1 bit each) | [31:0] (32 independent 1-bit fields) | `0b0` (all bits) | RW *(literal, §1.4 — no W1C encoding declared)* | Default | R / W |

All 576 fields verified programmatically to be individually declared with `reset="0b0"`, `sw_access="RW"`, `hw_access="Default"`, `read_behavior="R"`, `write_behavior="W"`, `write_enable=""`, `unlock_key=""`, `external_reset_source=""`, `presence_condition=""`, `enumerated_values=[]`. Field-level `description` for each bit equals its own name (e.g., `"description": "group_status_bit31"`), i.e. no interrupt-ID-to-bit mapping or additional semantic description is present beyond the bit name and position.

All fields across all 27 registers: `external_reset_source = ""` (no distinct external reset source declared — default/single reset domain applies), `write_enable = ""` (no conditional write-enable gating declared), `unlock_key = ""` (no write-lock/unlock mechanism declared), `presence_condition = ""` (unconditionally present), `enumerated_values = []` (no enumerated legal values declared for any field across all 626 fields), `user_defined_properties = {}`.

---

## 3. Verification Objectives

Based solely on the RegMan content, the verification of GIC600 (`gic600_1`) registers shall demonstrate that:

1. **OBJ-1**: Every declared register (27 total) is correctly accessible at its declared offset via the `ACELite_Slave` frontdoor interface, with the declared 32-bit register access width.
2. **OBJ-2**: Every field behaves per its declared `sw_access`/`read_behavior`/`write_behavior` — RO (`GICD_TYPER`, `GICD_IIDR`, plus `RWP`/`DS` in `GICD_CTLR`), RW (21 registers and their fields, including literal-RW reserved bits per §1.4), WO (`GICD_SETSPI_NSR`, `GICD_CLRSPI_NSR`, `GICD_SETSPI_SR`, `GICD_CLRSPI_SR`).
3. **OBJ-3**: All registers reset to their declared reset value following a DUT reset, as computed in §2.1/§2.2 (e.g. `GICD_CTLR = 0x00000070`, `GICD_TYPER = 0x00790006`, `GICD_IIDR = 0x0201743B`, all 18 bit-array registers `= 0x00000000`).
4. **OBJ-4**: The RAL model's mirror value tracks the DUT register value correctly across frontdoor writes, reads, and backdoor peek/poke operations (mirror/desired/actual consistency), for all 27 registers.
5. **OBJ-5**: `GICD_TYPER` and `GICD_IIDR` are verified as fully RO — software writes of any value do not alter their state, consistent with declared `sw_access: RO`/`write_behavior: NA`/`hw_access: RO`.
6. **OBJ-6**: `GICD_SETSPI_NSR`, `GICD_CLRSPI_NSR`, `GICD_SETSPI_SR`, `GICD_CLRSPI_SR` are verified as fully WO — software reads return a defined value per RAL (not a functional-state readback, since `read_behavior: NA` is declared for their fields) and do not error.
7. **OBJ-7**: `RWP` and `DS` in `GICD_CTLR` are individually verified as RO sub-fields within an otherwise RW register (mixed-access register verification).
8. **OBJ-8**: All 576 fields of `GICD_IGROUPR1`–`6`, `GICD_ISENABLER1`–`6`, `GICD_ICENABLER1`–`6` are verified as independent RW bits (bit-bash / walking-1s-and-0s), per the consolidated approach of §1.4/§2.3.2.
9. **OBJ-9**: Accesses that violate the declared access policy (writes to RO registers/fields; reads to WO registers, where meaningful) do not alter DUT state / do not return functionally-meaningful data, consistent with declared `sw_access`.
10. **OBJ-10 (Information Required)**: The actual hardware-side update behavior implied by `hw_access: "Default"` for the 610 non-pure-RO fields — pending clarification per §1.4.
11. **OBJ-11 (Information Required)**: Whether `GICD_ICENABLER1`–`6` implement write-1-to-clear semantics in RTL despite no such encoding in the RegMan — pending clarification per §1.4.
12. **OBJ-12 (Information Required)**: Expected access behavior (error response vs. RAZ/WI vs. undefined) for the unmapped offset ranges identified in §2.2, given `error_response: true` at the bus-interface level — pending clarification.

---

## 4. Verification Strategy

### 4.1 Frontdoor Verification Strategy

- All 27 registers shall be accessed via the UVM RAL frontdoor path over the `ACELite_Slave` bus, exercising the RAL-generated `reg_block` for `gic600_1`.
- Each register/field shall be written (where `sw_access` permits — RW and WO) and read (where `sw_access` permits — RW and RO) through the RAL `write()`/`read()` sequence API, and results checked against the RAL mirror.
- `GICD_TYPER` and `GICD_IIDR` shall be exercised via frontdoor **read only**, consistent with declared `RO` access.
- `GICD_SETSPI_NSR`, `GICD_CLRSPI_NSR`, `GICD_SETSPI_SR`, `GICD_CLRSPI_SR` shall be exercised via frontdoor **write only** for functional checks, consistent with declared `WO` access (see §4.5 for negative-read handling).
- All 18 bit-array registers (`GICD_IGROUPR*`, `GICD_ISENABLER*`, `GICD_ICENABLER*`) shall additionally be exercised with UVM RAL's built-in `uvm_reg_bit_bash_seq` to independently toggle and verify each of their 32 bits per register.
- Frontdoor tests shall use the declared 32-bit register access width for all 27 registers (bus data width is 64 bits per `ACELite_Slave`/`ACELite_Slave_MM`, but no register exceeds 32 bits — sub-64-bit-bus access handling is therefore applicable; see Appendix A for open items on byte/partial access).

### 4.2 Backdoor Verification Strategy

- All 27 registers shall be exercised via RAL `peek()`/`poke()` backdoor access, where a backdoor HDL path is bound in the RAL model.
- For `GICD_TYPER`/`GICD_IIDR` (RO), backdoor `poke()` is the only means to change the RAL-mirrored value for verifying that frontdoor `read()` correctly reflects a poked value, since no frontdoor write path exists.
- For the 4 WO registers, backdoor `peek()` is used to confirm that a frontdoor `write()` reached the underlying storage (used in conjunction with mirror checks, since frontdoor read of `read_behavior: NA` fields is not functionally meaningful).
- For the 21 RW registers (including the 18 bit-array registers and all literal-RW reserved bits per §1.4), backdoor peek/poke shall be cross-checked against frontdoor write/read for full consistency.

### 4.3 Register Reset Verification

- All 27 registers shall be verified to reset to their RegMan-declared value (§2.1/§2.2) following DUT reset, using RAL's `uvm_reg_hw_reset_seq` and/or explicit post-reset `read()`/`peek()` + mirror-compare.
- Reset verification shall be performed both from a "clean" (post-elaboration) reset and from a reset applied after prior register writes, to confirm registers do not retain stale values.
- Mixed-access registers (`GICD_CTLR`, whose `RWP`/`DS` bits are RO within an otherwise-RW register) shall have every constituent bit checked individually against its own declared reset value.

### 4.4 Register Access Policy Verification

- RO registers/fields (`GICD_TYPER`, `GICD_IIDR`, `RWP`, `DS`): verify that frontdoor writes of arbitrary/random/all-ones data produce **no change** to the RAL mirror or DUT state.
- WO registers (`GICD_SETSPI_NSR`, `GICD_CLRSPI_NSR`, `GICD_SETSPI_SR`, `GICD_CLRSPI_SR`): verify frontdoor writes succeed without bus error (`error_response` path not triggered for legitimate WO writes); verify frontdoor reads do not report a functionally meaningful register value (`read_behavior: NA`), per §4.5 for the exact expected read response.
- RW registers/fields (21 registers, including literal-RW reserved bits and all 576 bit-array fields): verify full read/write access at every declared bit position.
- `GICD_ICENABLER1`–`6`: verified strictly as literal generic RW per §1.4 — **no W1C-specific access-policy test** is included; the discrepancy versus conventional GICv3 ICENABLER semantics is tracked as **Information Required** (Appendix A), not tested.

### 4.5 Negative Testing Strategy

- **RO write rejection**: Attempt frontdoor writes to `GICD_TYPER`, `GICD_IIDR`, and the `RWP`/`DS` bits of `GICD_CTLR` with randomized/all-ones patterns; verify no state change.
- **WO read handling**: Attempt frontdoor reads of `GICD_SETSPI_NSR`, `GICD_CLRSPI_NSR`, `GICD_SETSPI_SR`, `GICD_CLRSPI_SR`; verify the RAL/bus does not report an error and the read completes per the declared `read_behavior: NA` (exact expected read data value is **Information Required** — not declared in the RegMan; see Appendix A).
- **Unmapped-offset access** *(Information Required)*: Frontdoor access to any offset in the gaps identified in §2.2 (`0x00C–0x01F`, `0x028–0x03F`, `0x044–0x047`, `0x04C–0x04F`, `0x054–0x057`, `0x05C–0x083`, `0x09C–0x103`, `0x11C–0x183`, `0x19C–0x1FF`) — expected response (bus error given `error_response: true`, RAZ/WI, or other) is not declared in the RegMan and is pending clarification before this test can be assigned pass/fail criteria.
- **Bit-array boundary/negative patterns**: For all 18 bit-array registers, walking-1s, walking-0s, all-ones, and all-zeros patterns shall be applied and read back, in addition to per-bit bash, to catch bit-to-bit aliasing within a register.

---

## 5. Verification Features

Feature IDs are of the form `FT-<REGISTER>-<NNN>`. Per §1.4, the 18 bit-array registers use one consolidated feature-set per register group rather than one row per bit.

### 5.1 Register: `GICD_CTLR` (offset `0x0`, 32-bit, RW with RO sub-fields)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-CTLR-001 | Frontdoor write/read of all RW bits (`RESERVED1[30:8]`, `E1NWF`, `ARE_NS`, `ARE_S`, `RESERVED0`, `EnableGrp1_s`, `EnableGrp1_ns`, `EnableGrp0`) | RAL frontdoor `write()`/`read()` | Read-back matches last written value for every RW bit position |
| FT-CTLR-002 | RO sub-field write rejection (`RWP[31]`, `DS[6]`) | Frontdoor `write()` of full register with RO bits set/cleared, then `read()` | `RWP` and `DS` retain their prior value; only RW bits change |
| FT-CTLR-003 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000070` (`DS=1`, `ARE_NS=1`, `ARE_S=1`, all other bits `0`) |
| FT-CTLR-004 | Backdoor peek/poke consistency (RW bits) | RAL `peek()`/`poke()` | Poked RW-bit values observable via subsequent frontdoor read |
| FT-CTLR-005 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-CTLR-006 | Boundary value test | Write `0x00000000`, `0xFFFFFFFF`, walking-1/walking-0 across RW bits | RW bits reflect pattern exactly; RO bits (`RWP`, `DS`) remain unaffected by the write |
| FT-CTLR-007 *(literal-RW reserved bit, §1.4)* | `RESERVED1[30:8]`, `RESERVED0[3]` full RW verification | Frontdoor write/read, boundary values | Behaves as plain RW per literal declaration; no SBZP/reserved special-casing applied |

### 5.2 Register: `GICD_TYPER` (offset `0x4`, 32-bit, RO)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-TYPER-001 | Frontdoor read of full 32-bit register | RAL frontdoor `read()` | Read returns `0x00790006` post-reset (`ITLinesNumber=0b110`, `MBIS=1`, `IDbits=0b1111`, all else `0`) |
| FT-TYPER-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00790006` |
| FT-TYPER-003 | RO negative-write test | Frontdoor `write()` with all-ones/randomized pattern, then `read()` | Register value unchanged by the write attempt |
| FT-TYPER-004 | Backdoor poke / frontdoor read consistency | RAL `poke()`, then frontdoor `read()` | Frontdoor read reflects the poked value |
| FT-TYPER-005 | Mirror/desired/actual consistency | RAL `mirror()` after backdoor poke | RAL mirror matches DUT actual value |

### 5.3 Register: `GICD_IIDR` (offset `0x8`, 32-bit, RO)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-IIDR-001 | Frontdoor read of full 32-bit register | RAL frontdoor `read()` | Read returns `0x0201743B` post-reset (`ProductID=0x02`, `Variant=0x1`, `Revision=0x7`, `Implementer=0x43B`) |
| FT-IIDR-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x0201743B` |
| FT-IIDR-003 | RO negative-write test | Frontdoor `write()` with all-ones/randomized pattern, then `read()` | Register value unchanged by the write attempt |
| FT-IIDR-004 | Backdoor poke / frontdoor read consistency | RAL `poke()`, then frontdoor `read()` | Frontdoor read reflects the poked value |
| FT-IIDR-005 | Mirror/desired/actual consistency | RAL `mirror()` after backdoor poke | RAL mirror matches DUT actual value |

### 5.4 Register: `GICD_FCTLR` (offset `0x20`, 32-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-FCTLR-001 | Frontdoor write/read of all fields (`POS`, `QDENY`, `DCC`, `NSACR[17:16]`, `CGO[12:4]`, `SIP`, plus literal-RW reserved bits `RESERVED0`–`RESERVED4`) | RAL frontdoor `write()`/`read()` | Read-back matches last written 32-bit value |
| FT-FCTLR-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000000` |
| FT-FCTLR-003 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read |
| FT-FCTLR-004 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-FCTLR-005 | Boundary value test | Write `0x0`, `0xFFFFFFFF`, walking-1/walking-0 patterns | All patterns read back exactly as written (full 32-bit RW, no SBZP masking declared) |

### 5.5 Register: `GICD_SAC` (offset `0x24`, 32-bit, RW)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-SAC-001 | Frontdoor write/read of all fields (`GICPNS`, `GICTNS`, `DSL`, plus literal-RW `RESERVED0[31:3]`) | RAL frontdoor `write()`/`read()` | Read-back matches last written 32-bit value |
| FT-SAC-002 | Reset value verification | Apply DUT reset; RAL `read()`/mirror check | Register reads `0x00000000` |
| FT-SAC-003 | Backdoor peek/poke consistency | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read |
| FT-SAC-004 | Mirror/desired/actual consistency | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-SAC-005 | Boundary value test | Write `0x0`, `0xFFFFFFFF`, walking-1/walking-0 patterns | All patterns read back exactly as written |

### 5.6–5.9 Registers: `GICD_SETSPI_NSR` (`0x40`), `GICD_CLRSPI_NSR` (`0x48`), `GICD_SETSPI_SR` (`0x50`), `GICD_CLRSPI_SR` (`0x58`) — all 32-bit, WO, identical field layout (`RESERVED0[31:10]`, `ID[9:0]`)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-SETNSR-001 / FT-CLRNSR-001 / FT-SETSR-001 / FT-CLRSR-001 | Frontdoor write of full register (`ID[9:0]` + literal-RW `RESERVED0[31:10]`) | RAL frontdoor `write()` | Write completes without bus error (`error_response` path not triggered) |
| FT-SETNSR-002 / FT-CLRNSR-002 / FT-SETSR-002 / FT-CLRSR-002 | WO read handling *(Information Required — expected read data not declared)* | Frontdoor `read()` after write | Read does not report a bus error; returned data value is **Information Required** (§4.5) |
| FT-SETNSR-003 / FT-CLRNSR-003 / FT-SETSR-003 / FT-CLRSR-003 | Reset value verification | Apply DUT reset; RAL mirror/backdoor `peek()` check | Underlying storage reads `0x00000000` (backdoor) post-reset |
| FT-SETNSR-004 / FT-CLRNSR-004 / FT-SETSR-004 / FT-CLRSR-004 | Backdoor peek consistency with frontdoor write | Frontdoor `write()`, then backdoor `peek()` | Peeked value matches last frontdoor-written value |
| FT-SETNSR-005 / FT-CLRNSR-005 / FT-SETSR-005 / FT-CLRSR-005 | Boundary value test (`ID[9:0]` = `0x000`, `0x3FF`, walking-1/walking-0) | Frontdoor `write()`, backdoor `peek()` | All patterns present exactly as written in underlying storage |

### 5.10 Register Group: `GICD_IGROUPR1`–`6` (offsets `0x84, 0x88, 0x8C, 0x90, 0x94, 0x98`, 32-bit, RW — 192 bits total, consolidated per §1.4/§2.3.2)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-IGRP-001 | Frontdoor write/read of full 32-bit register, per instance (×6) | RAL frontdoor `write()`/`read()` | Read-back matches last written 32-bit value, for each of the 6 registers |
| FT-IGRP-002 | Per-bit independence (bit-bash) — all 32 `group_status_bit<n>` fields per register | `uvm_reg_bit_bash_seq` / manual walking-1/walking-0 | Each of the 32 bits is independently settable/clearable with no aliasing to adjacent bits, for each of the 6 registers (192 bits total) |
| FT-IGRP-003 | Reset value verification, per instance (×6) | Apply DUT reset; RAL `read()`/mirror check | All 6 registers read `0x00000000` |
| FT-IGRP-004 | Backdoor peek/poke consistency, per instance (×6) | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read |
| FT-IGRP-005 | Mirror/desired/actual consistency, per instance (×6) | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-IGRP-006 | Boundary value test, per instance (×6) | Write `0x00000000`, `0xFFFFFFFF` | All patterns read back exactly as written |

### 5.11 Register Group: `GICD_ISENABLER1`–`6` (offsets `0x104, 0x108, 0x10C, 0x110, 0x114, 0x118`, 32-bit, RW — 192 bits total, consolidated per §1.4/§2.3.2)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-ISEN-001 | Frontdoor write/read of full 32-bit register, per instance (×6) | RAL frontdoor `write()`/`read()` | Read-back matches last written 32-bit value, for each of the 6 registers |
| FT-ISEN-002 | Per-bit independence (bit-bash) — all 32 `set_enable_bit<n>` fields per register | `uvm_reg_bit_bash_seq` / manual walking-1/walking-0 | Each of the 32 bits is independently settable/clearable with no aliasing, for each of the 6 registers (192 bits total) |
| FT-ISEN-003 | Reset value verification, per instance (×6) | Apply DUT reset; RAL `read()`/mirror check | All 6 registers read `0x00000000` |
| FT-ISEN-004 | Backdoor peek/poke consistency, per instance (×6) | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read |
| FT-ISEN-005 | Mirror/desired/actual consistency, per instance (×6) | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-ISEN-006 | Boundary value test, per instance (×6) | Write `0x00000000`, `0xFFFFFFFF` | All patterns read back exactly as written |

### 5.12 Register Group: `GICD_ICENABLER1`–`6` (offsets `0x184, 0x188, 0x18C, 0x190, 0x194, 0x198`, 32-bit, RW *(literal, §1.4)* — 192 bits total, consolidated per §1.4/§2.3.2)

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-ICEN-001 | Frontdoor write/read of full 32-bit register, per instance (×6) — verified strictly as literal generic RW | RAL frontdoor `write()`/`read()` | Read-back matches last written 32-bit value, for each of the 6 registers |
| FT-ICEN-002 | Per-bit independence (bit-bash) — all 32 `clear_enable_bit<n>` fields per register | `uvm_reg_bit_bash_seq` / manual walking-1/walking-0 | Each of the 32 bits is independently settable/clearable with no aliasing, for each of the 6 registers (192 bits total) |
| FT-ICEN-003 | Reset value verification, per instance (×6) | Apply DUT reset; RAL `read()`/mirror check | All 6 registers read `0x00000000` |
| FT-ICEN-004 | Backdoor peek/poke consistency, per instance (×6) | RAL `peek()`/`poke()` | Poked value observable via subsequent frontdoor read |
| FT-ICEN-005 | Mirror/desired/actual consistency, per instance (×6) | RAL `mirror()` after writes | RAL mirror matches DUT actual value |
| FT-ICEN-006 | Boundary value test, per instance (×6) | Write `0x00000000`, `0xFFFFFFFF` | All patterns read back exactly as written |
| FT-ICEN-007 *(Information Required)* | Write-1-to-clear (W1C) semantics vs. `GICD_ISENABLER*` interaction | Not determinable from RegMan | No W1C encoding, and no cross-register (`ISENABLER`↔`ICENABLER`) interaction attribute is declared anywhere in the file; per §1.4, not tested — pending RTL/architecture confirmation |

### 5.13 Address Map Features

| Feature ID | Description | Verification Method | Expected Result |
|---|---|---|---|
| FT-MAP-001 | Correct address decode for all 27 registers | Frontdoor access at each declared offset (§2.1) | Access reaches the intended register only; no aliasing to adjacent/unmapped space |
| FT-MAP-002 *(Information Required)* | Unmapped-offset access behavior (§2.2 gaps) | Frontdoor access to offsets outside the 27 declared registers, within the `0x200` window | RegMan does not define expected response (error vs. RAZ/WI vs. default read); pending clarification |
| FT-MAP-003 | Bus error-response capability | Negative-path frontdoor access expected to trigger bus-level error | `error_response: true` declared at bus-interface level — exact triggering conditions beyond unmapped access are not specified; scope limited to what RegMan declares |

---

## 6. Test Plan

| Test ID | Test Name | Applicable Register(s) | Category | Description |
|---|---|---|---|---|
| TC-001 | `gicd_ctlr_frontdoor_rw` | GICD_CTLR | Frontdoor R/W | Write/read RW bits of GICD_CTLR via frontdoor; verify RO bits (`RWP`, `DS`) unaffected |
| TC-002 | `gicd_typer_frontdoor_ro_read` | GICD_TYPER | Frontdoor R/W | Frontdoor read of GICD_TYPER; verify against reset value / backdoor-poked value |
| TC-003 | `gicd_iidr_frontdoor_ro_read` | GICD_IIDR | Frontdoor R/W | Frontdoor read of GICD_IIDR; verify against reset value / backdoor-poked value |
| TC-004 | `gicd_fctlr_frontdoor_rw` | GICD_FCTLR | Frontdoor R/W | Write/read GICD_FCTLR via frontdoor |
| TC-005 | `gicd_sac_frontdoor_rw` | GICD_SAC | Frontdoor R/W | Write/read GICD_SAC via frontdoor |
| TC-006 | `spi_wo_regs_frontdoor_write` | GICD_SETSPI_NSR, GICD_CLRSPI_NSR, GICD_SETSPI_SR, GICD_CLRSPI_SR | Frontdoor R/W | Frontdoor write of `ID[9:0]` to each WO register; confirm via backdoor peek |
| TC-007 | `igroupr_frontdoor_rw` | GICD_IGROUPR1–6 | Frontdoor R/W | Write/read full 32-bit value to each of the 6 IGROUPR instances |
| TC-008 | `isenabler_frontdoor_rw` | GICD_ISENABLER1–6 | Frontdoor R/W | Write/read full 32-bit value to each of the 6 ISENABLER instances |
| TC-009 | `icenabler_frontdoor_rw` | GICD_ICENABLER1–6 | Frontdoor R/W | Write/read full 32-bit value to each of the 6 ICENABLER instances (literal RW, §1.4) |
| TC-010 | `all_regs_backdoor_peek_poke` | All 27 registers | Backdoor | RAL `peek()`/`poke()` on every register; cross-check with frontdoor where access permits |
| TC-011 | `all_regs_reset_value_check` | All 27 registers | Reset | Apply reset; verify each register equals its declared reset value (§2.1) |
| TC-012 | `ro_regs_negative_write` | GICD_TYPER, GICD_IIDR, GICD_CTLR (`RWP`,`DS` bits) | Access Permission (RO) | Attempt frontdoor write with all-ones/randomized pattern to RO registers/bits; verify no change |
| TC-013 | `wo_regs_read_handling` | GICD_SETSPI_NSR, GICD_CLRSPI_NSR, GICD_SETSPI_SR, GICD_CLRSPI_SR | Access Permission (WO) *(Information Required, §4.5)* | Frontdoor read after write; verify no bus error; returned-data expectation pending clarification |
| TC-014 | `rw_regs_field_rw_check` | GICD_CTLR (RW bits), GICD_FCTLR, GICD_SAC, GICD_IGROUPR1–6, GICD_ISENABLER1–6, GICD_ICENABLER1–6 | Access Permission (RW) | Verify every RW field (including literal-RW reserved bits, §1.4) is fully read/write |
| TC-015 | `bit_array_bit_bash` | GICD_IGROUPR1–6, GICD_ISENABLER1–6, GICD_ICENABLER1–6 | Random Access / Bit-Bash | `uvm_reg_bit_bash_seq` (or equivalent) exercising all 32 bits per register independently, for all 18 registers (576 bits total) |
| TC-016 | `all_regs_mirror_update_check` | All 27 registers | Mirror/Update | RAL `mirror()`/`update()` calls after writes/pokes; confirm no mismatch reported |
| TC-017 | `ral_built_in_hdl_reg_access_seq` | All 27 registers | Random Access | Standard UVM RAL built-in sequences (`uvm_reg_hw_reset_seq`, `uvm_reg_bit_bash_seq`, `uvm_reg_access_seq`) applied to all registers |
| TC-018 | `rw_wo_regs_boundary_values` | All RW and WO registers (25 of 27; excludes 2 RO) | Boundary | Write `0x0`, `0xFFFFFFFF` (or field-width-masked equivalent), walking-1/walking-0 patterns; verify exact read-back / backdoor peek |
| TC-019 *(Information Required)* | `unmapped_offset_access` | Address map | Negative | Access offsets in the gaps identified in §2.2 — expected response pending clarification |
| TC-020 *(Information Required)* | `icenabler_w1c_semantics_check` | GICD_ICENABLER1–6 | Negative / Architecture Confirmation | Not authored per §1.4 — tracked as a pending item to confirm actual RTL W1C behavior against the literal-RW RegMan declaration before deciding whether this test is needed |

---

## 7. Functional Coverage Plan

| Coverage Group | Coverpoints | Source |
|---|---|---|
| `cg_register_access` | Per-register: read hit, write hit (write hit N/A for `GICD_TYPER`/`GICD_IIDR`; read hit flagged Information Required for the 4 WO registers per §4.5), for all 27 registers | Register list, §2.1 |
| `cg_ctlr_bits` | Individual coverpoints for `RWP`, `E1NWF`, `DS`, `ARE_NS`, `ARE_S`, `EnableGrp1_s`, `EnableGrp1_ns`, `EnableGrp0` (set/clear) | §2.3.1, §5.1 |
| `cg_typer_iidr_readback` | Coverpoint: read returns exact declared reset-derived value (`0x00790006` / `0x0201743B`) | §2.1, §5.2/§5.3 |
| `cg_fctlr_sac_values` | Value bins per RW field: `0`, all-ones (field-width-masked), min/max boundary | §5.4/§5.5, TC-018 |
| `cg_spi_id_field` | `ID[9:0]` value bins: `0x000`, `0x3FF`, min/max boundary, random mid-range — crossed across the 4 WO registers | §5.6–5.9, TC-006 |
| `cg_bitarray_bitbash` | Per-bit set/clear coverage for all 32 bits, crossed across all 6 instances, for each of `IGROUPR`, `ISENABLER`, `ICENABLER` (3 × 6 × 32 = 576 coverpoints) | §5.10–5.12, TC-015 |
| `cg_access_type` | Frontdoor read, frontdoor write (where applicable), backdoor peek, backdoor poke — crossed with each register | §4.1, §4.2 |
| `cg_reset_coverage` | Reset applied + register read = expected reset value, per register (27 registers) | §4.3 |
| `cg_access_policy` | RO-register/bit write-attempt (no effect) vs. RW-register write (effective) vs. WO-register read (Information Required) | §4.4 |
| `cg_unmapped_access` *(Information Required)* | Access-outside-map coverpoint — bin definition depends on clarified expected behavior (§2.2) | Pending |
| `cg_icenabler_w1c` *(Information Required)* | W1C-specific coverage bins — not definable since no W1C behavior is declared (§1.4) | Pending |

> No RegMan-declared `enumerated_values` exist for any of the 626 fields (all are `[]`), so no enumerated-value coverage bins are defined; only reset/boundary/bit-bash/random value bins are applicable.

---

## 8. Requirements Traceability Matrix (RTM)

| Req ID | Requirement | Register/Field | Planned Test(s) | Verification Method |
|---|---|---|---|---|
| REQ-01 | `GICD_CTLR` accessible at offset 0x0, 32-bit, mixed RW/RO | GICD_CTLR | TC-001, TC-011, TC-012, TC-014 | Frontdoor, Reset, Access Policy |
| REQ-02 | `GICD_CTLR` resets to 0x00000070 | GICD_CTLR | TC-011 | Reset |
| REQ-03 | `GICD_CTLR.RWP`/`DS` reject software writes | GICD_CTLR | TC-012 | Access Policy — Negative |
| REQ-04 | `GICD_TYPER` accessible at offset 0x4, 32-bit RO | GICD_TYPER | TC-002, TC-011 | Frontdoor, Reset |
| REQ-05 | `GICD_TYPER` resets to 0x00790006 | GICD_TYPER | TC-011 | Reset |
| REQ-06 | `GICD_TYPER` rejects software writes | GICD_TYPER | TC-012 | Access Policy — Negative |
| REQ-07 | `GICD_IIDR` accessible at offset 0x8, 32-bit RO | GICD_IIDR | TC-003, TC-011 | Frontdoor, Reset |
| REQ-08 | `GICD_IIDR` resets to 0x0201743B | GICD_IIDR | TC-011 | Reset |
| REQ-09 | `GICD_IIDR` rejects software writes | GICD_IIDR | TC-012 | Access Policy — Negative |
| REQ-10 | `GICD_FCTLR` accessible at offset 0x20, 32-bit RW | GICD_FCTLR | TC-004, TC-011, TC-014 | Frontdoor, Reset, Access Policy |
| REQ-11 | `GICD_FCTLR` resets to 0x00000000 | GICD_FCTLR | TC-011 | Reset |
| REQ-12 | `GICD_SAC` accessible at offset 0x24, 32-bit RW | GICD_SAC | TC-005, TC-011, TC-014 | Frontdoor, Reset, Access Policy |
| REQ-13 | `GICD_SAC` resets to 0x00000000 | GICD_SAC | TC-011 | Reset |
| REQ-14 | 4 SPI WO registers accessible at their declared offsets, 32-bit WO | GICD_SETSPI_NSR, GICD_CLRSPI_NSR, GICD_SETSPI_SR, GICD_CLRSPI_SR | TC-006, TC-011 | Frontdoor, Reset |
| REQ-15 | 4 SPI WO registers reset to 0x00000000 | GICD_SETSPI_NSR, GICD_CLRSPI_NSR, GICD_SETSPI_SR, GICD_CLRSPI_SR | TC-011 | Reset |
| REQ-16 (Information Required) | Expected read-data behavior for WO registers confirmed | GICD_SETSPI_NSR, GICD_CLRSPI_NSR, GICD_SETSPI_SR, GICD_CLRSPI_SR | TC-013 | Access Policy — pending clarification |
| REQ-17 | `GICD_IGROUPR1`–`6` accessible, 32-bit RW, all 192 bits independently RW | GICD_IGROUPR1–6 | TC-007, TC-011, TC-014, TC-015 | Frontdoor, Reset, Access Policy, Bit-Bash |
| REQ-18 | `GICD_IGROUPR1`–`6` reset to 0x00000000 | GICD_IGROUPR1–6 | TC-011 | Reset |
| REQ-19 | `GICD_ISENABLER1`–`6` accessible, 32-bit RW, all 192 bits independently RW | GICD_ISENABLER1–6 | TC-008, TC-011, TC-014, TC-015 | Frontdoor, Reset, Access Policy, Bit-Bash |
| REQ-20 | `GICD_ISENABLER1`–`6` reset to 0x00000000 | GICD_ISENABLER1–6 | TC-011 | Reset |
| REQ-21 | `GICD_ICENABLER1`–`6` accessible, 32-bit, literal RW per §1.4, all 192 bits independently RW | GICD_ICENABLER1–6 | TC-009, TC-011, TC-014, TC-015 | Frontdoor, Reset, Access Policy, Bit-Bash |
| REQ-22 | `GICD_ICENABLER1`–`6` reset to 0x00000000 | GICD_ICENABLER1–6 | TC-011 | Reset |
| REQ-23 (Information Required) | `GICD_ICENABLER1`–`6` W1C semantics confirmed against RTL/architecture | GICD_ICENABLER1–6 | TC-020, FT-ICEN-007 | Not testable from RegMan alone — pending architecture/RTL confirmation |
| REQ-24 | RAL mirror tracks DUT state across all 27 registers | All 27 registers | TC-016, TC-017 | Mirror/Update |
| REQ-25 | Backdoor peek/poke consistent with frontdoor state | All 27 registers | TC-010 | Backdoor |
| REQ-26 (Information Required) | Unmapped address-space access behavior defined | Address map (0x200 window) | TC-019 | Negative — pending clarification |
| REQ-27 (Information Required) | Actual hardware-update behavior for `hw_access: "Default"` fields confirmed | All fields except pure-RO fields in GICD_TYPER/GICD_IIDR | (backdoor/mirror subset of TC-010, TC-016) | Not testable from RegMan alone — pending architecture spec |

---

## 9. Regression Plan

- **Regression Scope**: All tests in §6 (TC-001 through TC-018) form the baseline regression suite for every code/RTL change affecting `gic600_1`, using the RAL model generated from `GIC600.regman` (rtl_1p0_release_regman variant).
- **Regression Composition**:
  - Directed tests: TC-001–TC-009, TC-012, TC-013, TC-014 (deterministic per-register/per-field checks).
  - Backdoor/mirror consistency: TC-010, TC-016.
  - Reset: TC-011.
  - Built-in RAL sequences (random/generic) and bit-bash: TC-015, TC-017, run with multiple random seeds per regression cycle.
  - Boundary tests: TC-018, run with fixed corner-case values every regression, plus randomized mid-range values per seed.
- **Pass/Fail Criteria**: No RAL `UVM_ERROR`/`UVM_FATAL` on register mirror mismatch; all declared reset values, access policies, and per-bit bash results match exactly.
- **Exclusions Pending Clarification**: TC-019, TC-020 are **not** included in the baseline regression pass/fail gate until the "Information Required" items in §1.4, §2.2, §4.5 are resolved with the register/architecture owner. They are tracked as open items and should be added once clarified.
- **Seed/Iteration Policy**: Information Required — the RegMan does not define a required regression seed count, iteration count, or coverage closure threshold; this is a testbench/methodology decision outside the scope of the RegMan and should be defined by the verification team.
- **Re-run Triggers**: Any change to `GIC600.regman` (offsets, widths, access types, reset values, field count) should trigger regeneration of the RAL model and a full re-run of this regression suite.

---

## Appendix A: Open Items Requiring Clarification (Consolidated)

| # | Item | Section | Impact |
|---|---|---|---|
| 1 | Actual hardware-side update behavior implied by `hw_access: "Default"` on 610 of 626 fields | §1.4, §4.2, REQ-27 | Whether mirror-based frontdoor/backdoor test methodology (assumed as placeholder) remains valid |
| 2 | Whether `GICD_ICENABLER1`–`6` implement write-1-to-clear (W1C) semantics in RTL, despite being declared identically to `GICD_ISENABLER1`–`6` (plain RW) in the RegMan | §1.4, §5.12, TC-020, REQ-23 | Whether current literal-RW test treatment is correct, or whether dedicated W1C test cases and coverage bins must be added before sign-off |
| 3 | Expected behavior for accesses to unmapped offsets within the `0x200` address window (`0x00C–0x01F`, `0x028–0x03F`, `0x044–0x047`, `0x04C–0x04F`, `0x054–0x057`, `0x05C–0x083`, `0x09C–0x103`, `0x11C–0x183`, `0x19C–0x1FF`), given `error_response: true` at the bus-interface level | §2.2, §4.5, TC-019, REQ-26 | Negative test pass/fail criteria, functional coverage bins |
| 4 | Expected frontdoor read-data behavior for the 4 WO registers (`GICD_SETSPI_NSR`, `GICD_CLRSPI_NSR`, `GICD_SETSPI_SR`, `GICD_CLRSPI_SR`), whose fields declare `read_behavior: "NA"` | §4.5, §5.6–5.9, TC-013, REQ-16 | Whether a defined read-data expectation (e.g. all-zeros, last-written, or bus error) needs to be added to the WO test cases |
| 5 | Whether the numerous literal-`RW` `RESERVED*` bitfields (§1.4, §2.3.1) are intentionally software-writable in RTL, or whether the RegMan mis-declares access for what is architecturally a reserved/SBZP bit | §1.4, §2.3.1 | Whether literal-RW verification (current plan) or SBZP-style exclusion is the correct long-term test intent |
| 6 | Byte-enable / partial (sub-word) access restrictions, if any, given the 64-bit `ACELite_Slave` bus width vs. 32-bit register width | §4.1 | Whether partial-write negative tests are required |
| 7 | GIC-600 architecture specification / TRM revision this RegMan corresponds to (not stated in the file's `"description": "Structural stiched file"`) | §1.3 | Cross-reference of register semantics against the architectural specification for review purposes |
| 8 | Regression seed count / coverage closure threshold | §9 | Regression sizing and exit criteria |

This VPlan should be reviewed against the RTL/UVM regmodel and the GIC-600 architecture specification to resolve the items above prior to test implementation sign-off.
