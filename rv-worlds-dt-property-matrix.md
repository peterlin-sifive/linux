# RISC-V Worlds DT Property Matrix

Extension combinations and their corresponding device-tree property requirements,
derived from the RISC-V Worlds specification (`riscv-worlds/src/worlds.adoc`).

> **Note:** The column `Smwiddeleg` is the full spec name for what some documents
> abbreviate as `Smwdeleg`. Both refer to the same extension.

---

## Extension Combination Truth Table

| #   | Smwid | Smlwid | Smlwidlist | Smwiddeleg | Sswid | CSRs Present                                       |       `pmwid`       |    `pmwidlist`     |    `pmlwidlist`     | Notes                                                                                                            |
| --- | :---: | :----: | :--------: | :--------: | :---: | -------------------------------------------------- | :-----------------: | :----------------: | :-----------------: | ---------------------------------------------------------------------------------------------------------------- |
| 0   |   ❌   |   ❌    |     ❌      |     ❌      |   ❌   | *(none)*                                           | **REQ** (Smwid=off) | **NO** (Smwid=off) | **NO** (Smlwid=off) | Base: single fixed WID per hart, platform-defined only                                                           |
| 1   |   ✅   |   ❌    |     ❌      |     ❌      |   ❌   | `mwid`                                             |       **OPT**       |      **OPT**       | **NO** (Smlwid=off) | M-mode control only: lockable M-mode WID; all privilege modes share one WID                                      |
| 2   |   ❌   |   ✅    |     ❌      |     ❌      |   ❌   | `mlwid`                                            | **REQ** (Smwid=off) | **NO** (Smwid=off) |       **OPT**       | Lower-mode control: M-mode assigns WID to lower privileges; M-mode WID is platform-fixed                         |
| 3   |   ✅   |   ✅    |     ❌      |     ❌      |   ❌   | `mwid`, `mlwid`                                    |      **OPT**        |     **OPT**        |      **OPT**        | RoT M-mode control: RoT M-mode configures regular M-mode WID (lockable)                               |
| 4   |   ✅   |   ✅    |     ✅      |     ❌      |   ❌   | `mwid`, `mlwid`, `mlwidlist`                       |      **OPT**        |     **OPT**        |      **OPT**        | WID list restriction: M-mode can software-restrict the mlwid set; `mlwidlist` locks with `mwid`                  |
| 5   |   ❌   |   ✅    |     ❌      |     ✅      |   ✅   | `mlwid`, `mwiddeleg`, `slwid`                      | **REQ** (Smwid=off) | **NO** (Smwid=off) |      **OPT**        | S-delegated control (M-mode fixed): M-mode WID remains platform-fixed; S-mode controls U-mode WID via delegation |
| 6   |   ✅   |   ✅    |     ❌      |     ✅      |   ✅   | `mwid`, `mlwid`, `mwiddeleg`, `slwid`              |      **OPT**        |     **OPT**        |      **OPT**        | S-delegated control (M-mode programmable): M-mode WID is lockable; S-mode controls U-mode WID via delegation     |
| 7   |   ✅   |   ✅    |     ✅      |     ✅      |   ✅   | `mwid`, `mlwid`, `mlwidlist`, `mwiddeleg`, `slwid` |      **OPT**        |     **OPT**        |      **OPT**        | Complete feature set: all CSRs present; M-mode lockable WID, mlwidlist restriction, S-mode delegation            |

### Property Legend

DT validatator (Linux kernel DT bindings check) should raises error on violation of REQ and NO. 

| Symbol  | Meaning                                                                 |
| :-----: | ----------------------------------------------------------------------- |
| **REQ** | REQUIRED — must be present in DT                                        |
| **OPT** | OPTIONAL — may be present; provides additional platform constraint info |
| **NO**  | FORBIDDEN — must not be present (corresponding CSR does not exist)      |

---

## Key Dependencies

The following dependency rules govern which extension combinations are valid
(verified against spec §"RISC-V ISA World-ID Extensions"):

1. **Smlwid is independent of Smwid**
   > *"NOTE: Smlwid is not dependent on the Smwid extension."*
   — Smlwid may appear alone (Combo 2) or alongside Smwid (Combos 3–7).

2. **Smlwidlist requires both Smwid AND Smlwid**
   > *"The Smlwidlist extension depends on the Smwid and Smlwid extensions."*
   — Smlwidlist is invalid without both parents; it therefore only appears in Combos 4 and 7.

3. **Smwiddeleg/Sswid require Smlwid (but NOT Smwid or Smlwidlist)**
   > *"The Smwiddeleg extension requires the Smlwid extension."*
   > *"NOTE: these extensions are not dependent on Smwid nor Smlwidlist."*
   — Smwiddeleg (and the Sswid it enables) may appear without Smwid (Combo 5) or Smlwidlist (Combos 5–6).

4. **Sswid is conditionally enabled by Smwiddeleg**
   — Sswid (`slwid` CSR) is not an independently selectable extension; it is added to S-mode by
   Smwiddeleg when `mwiddeleg` is non-zero. Sswid never appears without Smwiddeleg.

---

## Property Logic

### `pmwid` — Platform-defined M-mode WID

*Spec reference: §"One world per hart (no ISA extension)" and §"One world per hart with the Smwid extension"*

| Smwid present? | Rule | Rationale |
|:--------------:|:----:|-----------|
| ❌ No | **REQUIRED** | With no `mwid` CSR, the hart's WID is entirely platform-defined (pinstrap, fuse, SoC register, etc.). The DT `pmwid` property is the **only** mechanism for software to discover the WID assigned to this hart. |
| ✅ Yes | **OPTIONAL** | The `mwid` CSR provides a readable view of the WID. `pmwid` describes the reset value shown by `mwid` out of reset; the platform may convey this via other means or software may read `mwid` directly. |

### `pmwidlist` — Platform-defined WID allowlist for `mwid`

*Spec reference: §"One world per hart with the Smwid extension"*

| Smwid present? | Rule | Rationale |
|:--------------:|:----:|-----------|
| ❌ No | **FORBIDDEN** | There is no `mwid` CSR; `pmwidlist` has no register to constrain and is meaningless without it. Including it would be misleading. |
| ✅ Yes | **OPTIONAL** | The platform *may* limit the set of WIDs assignable to `mwid` via a non-ISA mask. If so, `pmwidlist` encodes that mask in the DT so early-boot firmware can validate WID assignments without taking exceptions. |

### `pmlwidlist` — Platform-defined WID allowlist for `mlwid`

*Spec reference: §"Smlwid extension"*

| Smlwid present? | Rule | Rationale |
|:---------------:|:----:|-----------|
| ❌ No | **FORBIDDEN** | There is no `mlwid` CSR; `pmlwidlist` has no register to constrain and is meaningless without it. |
| ✅ Yes | **OPTIONAL** | The platform *may* limit the set of WIDs assignable to `mlwid` via a non-ISA mask. If Smlwidlist is also enabled, the `mlwidlist` CSR is initialized to `pmlwidlist` on reset and can only further restrict (never expand) the platform-allowed set. |

---

## CSR Summary

| CSR | Address | Mode | Introduced by | Description |
|-----|---------|:----:|---------------|-------------|
| `mwid` | `0x391` | M | Smwid | WID for M-mode; lockable via MSB lock bit |
| `mlwid` | `0x390` | M | Smlwid | WID assigned to lower-than-M privilege modes |
| `mlwidlist` | `0x749` | M | Smlwidlist | Software-configurable WID allowlist for `mlwid`; locked when `mwid` is locked |
| `mwiddeleg` | `0x748` | M | Smwiddeleg | Bitmask of WIDs delegated to S-mode for U-mode assignment |
| `slwid` | `0x190` | S | Sswid (via Smwiddeleg) | WID assigned to lower-than-S privilege modes; written by S-mode within `mwiddeleg` |
