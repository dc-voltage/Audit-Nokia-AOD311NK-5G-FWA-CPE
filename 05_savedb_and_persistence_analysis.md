# 05 — savedb and Persistence Analysis

## Phase 5 — Task: Determine Whether `savedb` Persists Configuration

### Evidence Investigation

| Investigation Step | Result | Confidence |
|---|---|---|
| **Locate `savedb` in binary** | Searched via `strings` and `grep` across assessment workspace | Found as a standalone string in `password_web_app.cgi` binary artifact |
| **Determine if `savedb` is a command, function, or identifier** | Checked across all available artifacts | Three possibilities remain: (1) a literal command string passed to a shell, (2) a function/symbol name, (3) an internal configuration-framework operation marker |
| **Cross-reference with `cfg_setObj()` and `SE_passwd_db_auth_hash_with_block()`** | Compared call order and return values in static analysis | All three strings appear in `password_web_app.cgi`; exact control flow not established without runtime testing |
| **Investigate persistence operation sequence** | Compared with previously observed 7-step flow | Step 6 in proposed model is "Persistence Through `savedb`"; exact invocation sequence unverified at runtime |
| **Determine flash vs. RAM persistence** | No runtime writes performed; no firmware dump available | Insufficient evidence; `savedb` may write to NVRAM, RAM, or be a no-op marker depending on implementation |
| **Check for multiple independent storage writes** | Compared with other CGI endpoints and `savedb` usage elsewhere in assessment | No other confirmed `savedb` invocations found in assessment evidence; password-change flow may use a single persistence call |

### Key Findings

| Finding | Classification | Evidence |
|---|---|---|
| **`savedb` string exists in `password_web_app.cgi`** | ✅ Binary-confirmed | `strings password_web_app.cgi | grep savedb` returns the string |
| **`savedb` appears in Step 6 of 7-step flow** | ✅ Source-confirmed | Earlier static-analysis reports (01-05) identify `savedb` as Step 6 |
| **Exact invocation mechanism unknown** | ⚠️ Inferred | `savedb` may be: (a) a function call, (b) a shell command, (c) a framework marker; control flow not established statically |
| **Whether `savedb` writes to flash/NVRAM** | ❓ Unknown without runtime testing | No firmware dump; no runtime writes performed; binary strings alone cannot establish persistence target |
| **No other `savedb` invocations found** | ✅ Observed | Across all assessment evidence, only one occurrence in `password_web_app.cgi` |
| **Persistence order relative to hashing** | ⚠️ Inferred from static placement | `savedb` appears after `SE_passwd_db_auth_hash_with_block()` in the proposed 7-step flow; exact ordering not verified |

### unresolved Questions

| Question | Missing Evidence |
|---|---|
| Is `savedb` a function call or command string? | Runtime disassembly or decompilation of `password_web_app.cgi` |
| Does `savedb` write to flash, NVRAM, or RAM? | Firmware analysis or runtime memory inspection |
| Is `savedb` followed by a commit/apply operation? | No observed `uci_commit()` or equivalent in assessment evidence |
| Does `savedb` failure propagate to the frontend? | No runtime password-change testing performed; no failure-state observation |

### Recommended Defensive Improvement (if deploying similar router firmware)

| Recommendation | Rationale |
|---|---|
| Document `savedb` invocation semantics explicitly | Avoids ambiguity between framework marker vs. functional call |
| Separate persistence from configuration-update logic | Makes it clear whether changes are written to non-volatile storage |
| Add explicit success/failure reporting for persistence step | Prevents misleading `{"ret":1}` responses when flash write fails |
| Include `cfg_commit()` or equivalent after `savedb` | Ensures runtime state and persistent storage are consistent |

### Summary

- `savedb` is a **confirmed string** present in `password_web_app.cgi`
- It appears in the **proposed 7-step password-change flow** as Step 6
- Its **exact nature** (function, command, or marker) and **persistence target** (flash/NVRAM/RAM) are **unknown** without runtime testing or full binary analysis
- No evidence of multiple `savedb` invocations or intermediate commit operations
- **Do not claim** that password changes are persisted to flash or that `savedb` guarantees non-volatile storage without additional authorized evidence
