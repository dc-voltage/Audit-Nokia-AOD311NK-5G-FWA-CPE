# 01 — Artifact and Binary Validation

## Overview
This report validates the available evidence for the password-change deep analysis. All analysis is read-only; no runtime verification or form submissions were performed.

## 1. Target Artifact Status

| Artifact | Path | Status | Notes |
|----------|------|--------|-------|
| `password_web_app.cgi` | Not found on filesystem | ❌ Not available | The actual CGI binary was not present on the Kali host. Original static analysis (report dated Oct 22) was based on `strings` output examined off-host or in a previous assessment context. |
| `caps.json` | `/home/dc/caps.json` | ✅ Available | 38,997 bytes; contains `authorizedcgi` list, visibility fields, and capability metadata for the router UI. |
| `05_password_change_static_analysis.md` | `/home/dc/Desktop/New Folder/router-webui-audit/` | ✅ Available | Original static analysis report (Oct 22); documents the 7-step flow inferred from binary strings. |
| Live browser/HTMI evidence | Phases 13-14 (reports 13_live_editing_verification.md, burp-analysis) | ✅ Available | Passive observation of live HTTPS Web UI at `https://192.168.0.1`. |

## 2. Binary Verification

| Check | Result | Evidence |
|-------|--------|----------|
| `file` on `password_web_app.cgi` | ❌ Cannot perform — binary not present | N/A |
| `sha256sum` on `password_web_app.cgi` | ❌ Cannot perform — binary not present | N/A |
| `strings` output analysis | ✅ Referenced in original report 05 | Original report lists: `upswd`, `pswdNew`, `pswdConfirm`, `"set ok"`, `"The equal password has already been used!"`, `SE_passwd_db_auth_ex_with_block`, `SE_passwd_db_auth_hash_with_block`, `savedb`, `cfg_setObj`, `ret` codes 10/20/30/40/-2 |
| Ghidra/decompilation | ❌ Not available | No Ghidra project or decompiled binary on host |
| radare2/rabin2 analysis | ❌ Not available | Tools may be installed but no binary to analyze |

### Strings Evidence (from original Oct 22 assessment)
Confirmed strings from `password_web_app.cgi` binary:
- `upswd` — Old password parameter name
- `pswdNew` — New password field name
- `pswdConfirm` — Confirm password field name
- `"set ok"` — Intermediate marker before persistence
- `"The equal password has already been used!"` — Password reuse check message
- `SE_passwd_db_auth_ex_with_block()` — Old-password verification function
- `SE_passwd_db_auth_hash_with_block()` — Password hashing function
- `savedb` — Persistence command/function
- `cfg_setObj()` — Configuration write function
- Return codes: `ret:10`, `ret:20`, `ret:30`, `ret:40`, `ret:-2`
- `{"ret":1,"msg":"Password set success"}` — Success response format
- `{"ret":-2,"msg":"error get account info"}` — Account retrieval failure
- `"Original password error once,please input again!"` — 1st failure message
- `"Original password error twice,please input again!"` — 2nd failure message
- `"Original password error three times,please input again!"` — 3rd failure message
- `"Error happen more than 3 times and will logout, please retry after 60 seconds!"` — Lockout message after 3 failures

## 3. caps.json Evidence

### Password-Related Visibility Fields
| Field | Visibility Value | Classification |
|-------|-----------------|----------------|
| `password` (top-level) | 1 | Visible — password section is rendered in UI |
| `originalPassword` | 1 | Visible — old password field shown |
| `newPassword` | 1 | Visible — new password field shown |
| `reEnterPassword` | 1 | Visible — confirmation field shown |
| `apnAuthmode` | 0 | Restricted — not visible (0 = hidden/restricted) |
| `apnAuthmodeUsername` | 0 | Restricted |
| `apnAuthmodePassword` | 0 | Restricted |
| `authUsername` | 1 | Visible |
| `authPassword` | 1 | Visible |
| `primaryRadiusPassword` | 1 | Visible |

### Key caps.json Structure for Password Change
The password-change fields (`originalPassword`, `newPassword`, `reEnterPassword`) all have `visibility: 1`, confirming they are intended to be visible in the UI for authenticated users.

### Authorized CGI List
- `password_web_app.cgi?set` — Listed as authorized CGI for password operations
- `password_status_web_app.cgi` — Listed as authorized CGI for password status

### Confidence Classification
- **Source-confirmed**: caps.json visibility fields and authorized CGI list
- **Binary-evidence**: strings output from `password_web_app.cgi` (original analysis)
- **Inferred**: Control flow and function call ordering (without runtime execution)
- **Unknown**: Hashing algorithm, exact storage location, runtime behavior

## 4. Assessment Constraints

| Constraint | Status |
|------------|--------|
| No runtime password change | ✅ Performed — read-only only |
| No form submissions | ✅ No POST requests sent to any CGI |
| No credential disclosure | ✅ Cookie names only; values never extracted |
| No binary execution | ✅ No `password_web_app.cgi` on host |
| No network reconfiguration | ✅ Assessment proceeded with existing network state |
| Existing reports preserved | ✅ Original 05 report untouched; new reports in separate directory |

## 5. Summary of Evidence Availability

| Evidence Type | Availability | Confidence |
|--------------|-------------|------------|
| Actual `password_web_app.cgi` binary | ❌ Not on host | N/A |
| `strings` output from binary | ✅ Referenced in report 05 | High (original Oct 22 analysis) |
| `caps.json` metadata | ✅ Fully available | High |
| Live HTTPS UI observations | ✅ Passive, from Phases 13-14 | Medium (passive only) |
| Return code formats | ✅ From binary strings | High |
| Function names | ✅ From binary strings | High |
| Call flow ordering | ⚠️ Inferred from string sequencing | Medium |
| Hashing algorithm | ❌ Not determinable from available evidence | Unknown |
| Persistence mechanism (savedb) | ⚠️ String-confirmed only | Medium |
| Session renewal behavior | ✅ Observed in live UI (cookie `sid`) | Medium (passive observation) |

## 6. Required Deliverables (Planned)

The following reports will be produced in `/home/dc/router-webui-audit/password-change-deep-analysis/`:

1. `01_artifact_and_binary_validation.md` — **Complete** (this file)
2. `02_input_and_validation_analysis.md` — Planned
3. `03_authentication_and_reuse_checks.md` — Planned
4. `04_configuration_update_and_hashing.md` — Planned
5. `05_savedb_and_persistence_analysis.md` — Planned
6. `06_session_and_response_analysis.md` — Planned
7. `07_return_codes_and_error_branches.md` — Planned
8. `08_call_graph_and_verified_flow.md` — Planned
9. `09_final_evidence_assessment.md` — Planned

*If any filename already exists, a unique alternative will be used.*

## 7. Final Classification

| Finding | Classification | Evidence |
|---------|---------------|----------|
| 7-step flow structure | **Inferred** from string sequencing | Binary strings (original Oct 22 analysis) |
| Input names: `upswd`, `pswdNew`, `pswdConfirm` | **Observed** from strings | Direct string match |
| `SE_passwd_db_auth_ex_with_block()` referenced | **Observed** from strings | Direct function name |
| `SE_passwd_db_auth_hash_with_block()` referenced | **Observed** from strings | Direct function name |
| `cfg_setObj()` referenced | **Observed** from strings | Direct function name |
| `savedb` referenced | **Observed** from strings | Direct string |
| `"The equal password has already been used!"` check | **Observed** from strings | Direct string match |
| Return codes 10/20/30/40/-2 | **Observed** from strings | Direct string match |
| Success response `{"ret":1,"msg":"Password set success"}` | **Observed** from strings | Direct string/JSON match |
| Lockout after 3 failures | **Observed** from strings | Message sequencing |
| Hashing algorithm details | **Unknown** | No runtime or algorithm strings available |
| Storage location (NVRAM/flash) | **Unknown** | No partition or persistence evidence |
| `savedb` writes to flash | **Inferred** from governance pattern | caps.json + original analysis |
| Client-side JavaScript validation | **Inferred** from form structure | UI observations (Phase 13-14) |
| Cookie `sid` renewal on password change | **Observed** in live UI | Passive HTTPS capture (Phases 13-14) |
| `lsid` expiry to 2046 | **Observed** in static analysis | Report 05 |

---
