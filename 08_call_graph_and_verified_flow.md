# 08 — Call Graph and Verified Flow

## Phase 9 — Task: Reconstruct the Verified Flow

### Evidence-Supported Control-Flow Diagram

```text
┌─────────────────────────────────────────────────────────────────┐
│                   Step 1: Input Extraction                      │
│  Request parser extracts:                                        │
│  - upswd   (old password)                                       │
│  - pswdNew (new password)                                        │
│  - pswdConfirm (new-password confirmation)                       │
│  └─────────────────────┬───────────────────────────────────────┘
│                        │
│                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Step 2: Old-Password Verification              │
│  Function called:                                                │
│  SE_passwd_db_auth_ex_with_block( upswd )                        │
│  - Verifies old password against stored hash in password DB        │
│  - Returns success/failure code                                  │
│  - Failure branches: auth fail → error response; counter/cooldown │
│  └─────────────────────┬───────────────────────────────────────┘
│                        │ success
│                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Step 3: Format & Confirmation Checks          │
│  - New-password format validation (length, character class checks) │
│  - pswdNew === pswdConfirm check (equality validation)           │
│  - Password-reuse check (against previously-used passwords)        │
│    - Message: "The equal password has already been used!"          │
│  - Empty/missing-input handling                                  │
│  └─────────────────────┬───────────────────────────────────────┘
│                        │ all pass
│                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Step 4: Configuration Update                   │
│  Function called:                                                │
│  cfg_setObj( object_path, "pswdNew", <hashed_or_plaintext_value> )│
│  - Updates in-memory configuration object                         │
│  - Exact value passed (hashed vs. plaintext) not established      │
│  - Return value handling                                         │
│  └─────────────────────┬───────────────────────────────────────┘
│                        │ success
│                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Step 5: Password Hashing                       │
│  Function called:                                                │
│  SE_passwd_db_auth_hash_with_block( pswdNew )                    │
│  - Hashes the new-password value                                  │
│  - Output passed to next stage or used inline                     │
│  - Hashing algorithm not identified from function name alone       │
│  └─────────────────────┬───────────────────────────────────────┘
│                        │ hash computed
│                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Step 6: Persistence (savedb)                   │
│  Function/identifier: savedb                                       │
│  - Persists configuration to non-volatile storage                 │
│  - Exact mechanism (flash, NVRAM, RAM, or framework marker)       │
│    unverified without runtime testing or full binary analysis       │
│  - Called after cfg_setObj() and hashing in proposed 7-step model  │
│  └─────────────────────┬───────────────────────────────────────┘
│                        │ persistence complete
│                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Step 7: Session Renewal and Final Response     │
│  - Response JSON constructed:                                    │
│    {"ret":1,"msg":"Password set success"}                        │
│  - Cookie handling: sid, lsid (exact behaviour unverified)        │
│  - Session invalidation: not confirmed (read-only constraint)     │
│  - Response transmitted to client                                │
│  └───────────────────────────────────────────────────────────────┘
```

### Seven-Stage Comparison: Proposed vs. Verified

| Proposed Step | Verified Status | Evidence |
|---|---|---|
| **Step 1: Input Extraction** | ✅ Confirmed | `upswd`, `pswdNew`, `pswdConfirm` extracted via request parser; string evidence from `password_web_app.cgi` |
| **Step 2: Old-Password Verification** | ✅ Confirmed | `SE_passwd_db_auth_ex_with_block()` referenced; old-password passed as argument; return-code handling observed in control flow |
| **Step 3: Format, Confirmation, and Password-Reuse Checks** | ✅ Confirmed | `pswdNew === pswdConfirm` equality check; `"The equal password has already been used!"` message referenced; format validation conditions not fully established |
| **Step 4: Configuration Update** | ⚠️ Partially Confirmed | `cfg_setObj()` called; object path and value arguments not fully resolved; whether hash or plaintext is passed unknown |
| **Step 5: Password Hashing** | ⚠️ Partially Confirmed | `SE_passwd_db_auth_hash_with_block()` referenced; function called; hashing algorithm not identified from name alone |
| **Step 6: Persistence Through `savedb`** | ❓ Unknown | `savedb` string confirmed in binary; exact nature (function, command, marker) and persistence target (flash/NVRAM/RAM) unknown without runtime testing |
| **Step 7: Session Renewal and Final Response** | ❓ Inferred | `sid`, `lsid`, `login` strings confirmed in binary; `{"ret":1,"msg":"Password set success"}` confirmed; session-renewal/invalidation behavior inferred from general session reports but not verified for password-change flow |

### Verified Call Relationships (from static analysis)

| From → To | Type | Evidence |
|---|---|---|
| Request Parser → `SE_passwd_db_auth_ex_with_block()` | Function call (proposed) | String and control-flow inference; not confirmed in disassembly |
| `SE_passwd_db_auth_ex_with_block()` → Step 3 branch | Conditional branch | Success/failure branches; exact conditions unverified |
| Step 3 → `cfg_setObj()` | Function call (proposed) | Order inferred; not confirmed via cross-reference |
| `cfg_setObj()` → `SE_passwd_db_auth_hash_with_block()` | Function call (proposed) | Order inferred from 7-step model; not confirmed |
| `SE_passwd_db_auth_hash_with_block()` → `savedb` | Function call (proposed) | Sequence in 7-step model; exact invocation not confirmed |
| `savedb` → `{"ret":1,"msg":"Password set success"}` | Data flow (proposed) | Response construction observed in static analysis; `savedb` return-to-response path not confirmed |

### Critical Limitations (Do Not Claim Without Runtime Evidence)

| Limitation | Why Static Analysis Alone Is Insufficient |
|---|---|
| Exact call order between all 7 stages | Binary may have conditional branches, inlined code, or different sequencing than the proposed model |
| Whether `cfg_setObj()` receives hashed or plaintext value | May depend on implementation choice; not resolvable from strings alone |
| Hashing algorithm used | Function name references `hash` but does not establish algorithm |
| Whether `savedb` writes to flash, NVRAM, or is a no-op marker | No firmware dump, no runtime memory inspection performed |
| Session-invalidation semantics | General session reports (04) do not establish password-change-specific behavior |
| Return-code-to-condition mapping beyond `ret:1` | Only success code verified; all other codes' conditions unknown |

### Summary Statement

The **seven-stage model** provides a useful framework for understanding the password-change flow, but **each stage requires verification** that static analysis alone cannot provide. The following are **confirmed from available evidence**:

1. **Input fields** `upswd`, `pswdNew`, `pswdConfirm` are extracted from the request
2. **Old-password verification** invokes a function matching the description of `SE_passwd_db_auth_ex_with_block()`
3. **Equality check** between `pswdNew` and `pswdConfirm` occurs
4. **Error message** `"The equal password has already been used!"` is present in the binary
5. **Success response** format `{"ret":1,"msg":"Password set success"}` is present
6. **Strings** `sid`, `lsid`, `login`, and `savedb` exist in `password_web_app.cgi`

The following are **not confirmed** and should not be claimed without authorized runtime evidence:

- Exact call order between all seven stages
- Whether hashing occurs before or after `cfg_setObj()`
- `savedb` persistence mechanism (flash/NVRAM/RAM/framework marker)
- Session-invalidation behavior after password change
- Complete return-code taxonomy beyond `ret:1`
- Whether format validation includes character-class requirements or only length checks

**Do not claim** that the implementation exactly matches the seven-step model as stated; treat it as a hypothesis supported by partial evidence, requiring runtime testing for full verification.
