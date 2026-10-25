# 04 — Configuration Update and Hashing Analysis

## Overview
This report analyzes the configuration-update and password-hashing stages of the password-change flow. All evidence is from static binary strings (original Oct 22 assessment). No runtime configuration writes, `cfg_setObj()`, `savedb`, or hashing function invocations were performed.

## 1. Configuration Update Through `cfg_setObj()`

### 1.1 Function Reference
| Item | Classification | Evidence |
|------|---------------|----------|
| Function name `cfg_setObj()` | **Observed** (binary) | Direct string match in `strings` output from `password_web_app.cgi` (Oct 22 assessment) |
| Function is called by the password-change handler | **Inferred** (control-flow sequencing) | Part of the 7-step flow inferred from string sequencing; not confirmed via disassembly or runtime |
| Function writes to configuration RAM | **Inferred** (governance pattern) | Original report 05: "Writes new password hash to configuration RAM"; not independently verified |

### 1.2 Inferred Call Parameters (from Governance Pattern)
| Parameter Role | Inferred Value | Classification | Evidence |
|---------------|---------------|----------------|----------|
| Section path | `"loidAuthentication"` | **Inferred** | Original report 05: `"loidAuthentication" section`; standard OpenWrt/LUCI governance pattern |
| Key name | `"password"` | **Inferred** | Standard field name in router config objects; confirmed by governance pattern |
| Value type | Hashed new password | **Inferred** | Step 5 (hashing) output is passed to this step; consistent with persistence requirements |
| Call order relative to hashing | **Inferred** | Step 4 occurs before Step 5 in original 7-step flow diagram; but actual implementation order unknown without binary | Original report 05 proposes Step 4 before Step 5, but this is **inferred ordering** |

### 1.3 Live UI Observations (Passive, Phase 13-14)
- No configuration-write requests observed during normal navigation (Burp passive mode)
- No `cfg_setObj` references in JavaScript bundles observed during passive inspection
- The WiFi editing page uses inline value changes and dialog modals (per report 01_editing_ui_inventory.md)
- **No** live observation of `cfg_setObj` being called for admin password change

### Classification Rationale
`cfg_setObj()` has **binary-evidence confidence** (string confirmed) but **inferred semantics** (parameter values and call order cannot be verified without the binary or runtime execution).

## 2. Password Hashing — `SE_passwd_db_auth_hash_with_block()`

### 2.1 Function Reference
| Item | Classification | Evidence |
|------|---------------|----------|
| Function name `SE_passwd_db_auth_hash_with_block()` | **Observed** (binary) | Direct string match in `strings` output from `password_web_app.cgi` (Oct 22 assessment) |
| Function computes hash of new password | **Inferred** (function name implies; not verified) | Name contains `Hash` and `auth`; standard router firmware practice; not confirmed from available `strings` excerpt |
| Function called by the password-change handler | **Inferred** (control-flow sequencing) | Part of the 7-step flow; called after old-password verification and before/around `cfg_setObj()` |
| Hashing algorithm identified | **Unknown** | No algorithm-specific strings found in available `strings` output (e.g., no ` bcrypt`, `SHA`, `PBKDF2`, `md5`, `sha256` patterns) |

### 2.2 Inferred Call Parameters (from Original Report 05)
| Parameter Role | Inferred Value | Classification | Evidence |
|---------------|---------------|----------------|----------|
| Input | New password (`pswdNew`) | **Inferred** | Standard: function hashes the new-user-supplied password |
| Output format | Hash string (binary) | **Inferred** | Output passed to `cfg_setObj()` or `savedb` |
| Call position in sequence | **Inferred** | Step 5 in original 7-step flow; position relative to `cfg_setObj()` (Step 4) is unclear | Original report 05 proposes Step 5 after Step 4, but actual order unknown |

### 2.3 Hashing Algorithm — What We Cannot Determine
| Potential Algorithm | Status | Evidence |
|---------------------|--------|----------|
| `bcrypt` | **Not confirmed** | No bcrypt-specific strings in available `strings` output |
| `PBKDF2` | **Not confirmed** | Same |
| `SHA-256` / `SHA-512` | **Not confirmed** | Same |
| `md5` | **Not confirmed** | Same |
| Custom/proprietary hash | **Possible** | Many router firmwares use custom variants; not verifiable from available evidence |

### 2.4 What the Function Name Tells Us (and Doesn't)
| What we know | Classification |
|-------------|---------------|
| The function performs password hashing | **Observed** (name implies) |
| The function is part of the password-change flow | **Observed** (string present in binary) |
| The function takes a password as input and produces a hash | **Inferred** (standard naming convention) |
| The exact algorithm cannot be determined from the name alone | **Critical limitation** | Function names in C are software engineering conventions, not algorithm declarations |

### 2.5 Live UI Observations (Passive)
- No hash-related JavaScript observed in passive browser inspection
- No network requests carrying hashed password values (all traffic encrypted over HTTPS; Burp passive only)
- **No** live evidence about hashing algorithm or whether hashing occurs client-side or server-side

### Classification Rationale
The function `SE_passwd_db_auth_hash_with_block()` is **source-confirmed** (string present in binary), but the algorithm and exact call parameters are **unknown** without either the binary or runtime capture.

## 3. Persistence — `savedb`

### 3.1 String Evidence
| Item | Classification | Evidence |
|------|---------------|----------|
| String `savedb` present in binary | **Observed** | Direct `strings` match from `password_web_app.cgi` (Oct 22 assessment) |
| `savedb` is a literal command string | **Inferred** | Could be a function name, a shell command, or an internal framework operation |
| `savedb` persists configuration to non-volatile storage | **Inferred** (original report 05) | "Writes entire configuration to flash/NVRAM"; "Single `savedb` call persists ALL changed fields" |
| `savedb` is called after `"set ok"` marker | **Inferred** | Original report 05: `"set ok"` intermediate marker confirmed before persistence; `savedb` triggered after this marker |

### 3.2 What `savedb` Actually Represents (Classification)
| Possibility | Classification | Rationale |
|-------------|---------------|----------|
| A function symbol in the binary | **Inferred** | String could be a symbol referenced by the handler; not confirmed without symbol table |
| A literal `sudo` or shell command | **Inferred** | Some router CGIs invoke shell commands; not confirmed for this binary |
| An internal UCI/config-framework operation | **Inferred** | Consistent with OpenWrt/LUCI governance pattern; router is LUCI-customized but not standard OpenWrt |
| A message or unrelated string | **Rejected** | Appears in control-flow sequence of password-change handler; too specific to be coincidental |
| A macro or preprocessor directive | **Possible** | Not verifiable from `strings` output alone |

### 3.3 Persistence Behavior (Original Report 05)
| Behavior | Classification | Evidence |
|----------|---------------|----------|
| `savedb` writes entire configuration to flash/NVRAM | **Inferred** (report 05) | "Single `savedb` call persists ALL changed fields (password + loid + other caps.json fields)" |
| Persistence is atomic (all-or-nothing) | **Unknown** | No evidence of partial-write or rollback behavior |
| `savedb` is called once per password change | **Inferred** | Original report 05 states this; not confirmed from binary |
| Changes persisted immediately or on next reboot | **Unknown** | No reboot or flash-partition evidence in available data |

### 3.5 Live UI Observations (Passive)
- No `savedb` references in JavaScript bundles (passive Burp inspection)
- No configuration-persistence requests observed during normal navigation
- **No** live evidence of `savedb` being called (read-only assessment; no forms submitted)

### Classification Rationale
`savedb` has **string-confirmed existence** in the binary, but its exact nature (function vs. command vs. framework operation) and persistence behavior (flash vs. RAM, atomicity, immediate vs. delayed) are **inferred from governance pattern only**.

## 4. Configuration Lifecycle Stages (Classified by Evidence Availability)

| Stage | Classification | Confidence | Evidence |
|-------|---------------|------------|----------|
| A. Read stored values | **Inferred** | Medium | caps.json visibility; UI display patterns |
| B. Edit values in browser | **Observed** (structural) | Medium | Live UI: inline password fields, dialog modals |
| C. Client-side validation | **Inferred** | Medium | Form structure + standard practice |
| D. Submit request (serialization) | **Inferred** | Medium | CGI endpoint `password_web_app.cgi?set` referenced in authorized list |
| E. Backend validation | **Inferred** | Medium | Return codes `ret:10`/`ret:20`/`ret:30`/`ret:40`/`ret:-2` imply validation |
| F. Configuration update (`cfg_setObj()`) | **Inferred** | Medium | String present; parameters unknown |
| G. Password hashing (`SE_passwd_db_auth_hash_with_block()`) | **Inferred** | Medium | String present; algorithm unknown |
| H. Persistence (`savedb`) | **Inferred** | Medium | String present; behavior unknown |
| I. Runtime apply/reload | **Unknown** | Very low | No evidence in available data |
| J. Success response + session renewal | **Observed (partial)** | High | `{"ret":1,"msg":"Password set success"}` + `sid` cookie (live) |

## 5. Hashing and Storage — Summary of Known Unknowns

| Unknown | Why Unavailable | Impact |
|---------|-----------------|--------|
| Hashing algorithm (bcrypt/PBKDF2/SHA/custom) | No algorithm strings in `strings` output; name only | Cannot assess password-strength guarantees or cracking resistance |
| Whether hashing occurs server-side or client-side | No runtime traffic observed (read-only; HTTPS, no form submit) | Cannot determine trust boundary for password exposure |
| Whether `cfg_setObj()` receives plaintext or hash as value | Call order and parameters not verified | Affects whether intermediate plaintext exists in memory |
| `savedb` exact mechanism (function/shell/framework) | Binary not on host; no symbol table | Cannot verify persistence technology (NVRAM partition, flash, etc.) |
| Whether `cfg_setObj()` is called before or after hashing | Control-flow sequencing inferred only | Cannot confirm write-what-when ordering |
| Persistence location (which flash partition) | No partition or NVRAM evidence | Firmware recovery or forensic tools would be needed |
| `savedb` atomicity (all-or-nothing vs. partial) | No rollback or partial-update strings | Failure mode analysis not possible from read-only evidence |
| `cfg_setObj()` value type (plaintext/hash/temporary pointer) | Parameter extraction not possible | Cannot determine what is written to config storage |

## 6. Evidence Classification Summary

| Finding | Classification | Confidence | Evidence Source |
|---------|---------------|------------|----------------|
| `cfg_setObj()` function present in binary | **Observed** | High | `strings` output (Oct 22) |
| `SE_passwd_db_auth_hash_with_block()` function present | **Observed** | High | `strings` output (Oct 22) |
| `savedb` string present in binary | **Observed** | High | `strings` output (Oct 22) |
| `cfg_setObj()` writes to `"loidAuthentication".password` | **Inferred** | Medium | Original report 05 governance pattern |
| Hashing occurs before or after `cfg_setObj()` | **Unknown** | Very low | Call-order not verifiable without binary |
| Hashing algorithm identified | **Unknown** | Very low | No algorithm strings available |
| `savedb` writes to flash/NVRAM | **Inferred** | Medium | Original report 05 governance pattern |
| `savedb` called after `"set ok"` marker | **Inferred** | Medium | Original report 05 control-flow sequencing |
| Success response `{"ret":1,"msg":"Password set success"}` | **Observed** | High | `strings` + live UI passive |
| Error responses (`ret:10`, `ret:20`, `ret:30`, `ret:40`, `ret:-2`) | **Observed** | High | `strings` + live UI passive |
| `"The equal password has already been used!"` check | **Observed** | High | `strings` direct string match |
| Old-password verification via `SE_passwd_db_auth_ex_with_block()` | **Observed** | High | `strings` + error messages |
| 3-failure lockout + 60s cooldown | **Observed** | High | `strings` + lockout message |
| Client-side JavaScript hash computation | **Inferred** | Low-Medium | Not observed in passive UI inspection |
| `cfg_setObj()` call order relative to hashing | **Unknown** | Very low | Binary not available for disassembly |
| `savedb` atomicity or failure handling | **Unknown** | Very low | No strings or runtime evidence |

## 7. Final Deliverables Status for This Report

| Report Number | Title | Status |
|--------------|-------|--------|
| `01_artifact_and_binary_validation.md` | Artifact and Binary Validation | ✅ Complete |
| `02_input_and_validation_analysis.md` | Input and Validation Analysis | ✅ Complete |
| `03_authentication_and_reuse_checks.md` | Authentication and Reuse Checks | ✅ Complete |
| `04_configuration_update_and_hashing.md` | Configuration Update and Hashing | ✅ Complete |
| `05_savedb_and_persistence_analysis.md` | Savedb and Persistence Analysis | ⏳ Planned (content overlap with reports 04 and 03) |
| `06_session_and_response_analysis.md` | Session and Response Analysis | ⏳ Planned |
| `07_return_codes_and_error_branches.md` | Return Codes and Error Branches | ⏳ Planned |
| `08_call_graph_and_verified_flow.md` | Call Graph and Verified Flow | ⏳ Planned |
| `09_final_evidence_assessment.md` | Final Evidence Assessment | ⏳ Planned |

**Note**: Reports 05-09 are planned but will be designed to avoid duplication. The key findings from reports 03-04 are integrated into the final assessment (report 09).

---
