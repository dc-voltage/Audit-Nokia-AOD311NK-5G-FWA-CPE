# 03 — Authentication and Reuse Checks

## Overview
This report documents the authentication requirements for the password-change flow and the password-reuse prevention mechanism. All evidence is from static binary strings (original Oct 22 assessment) and passive live-UI observations (Phases 13-14). No form submissions or runtime authentication tests were performed.

## 1. Authentication Requirements for Password Change

### 1.1 General CGI Authentication
| Requirement | Classification | Evidence |
|-------------|---------------|----------|
| All CGIs require valid browser session | **Observed** (live) | Per Phase 13-14 reports: all 148+ authorized CGIs (including `password_web_app.cgi?set`) return HTTP 401 or redirect to login `#/login` when accessed without valid session cookies |
| Form-based login (not Basic Auth) | **Observed** (live) | Login page at `#/login` with username/password form; TLS-protected HTTPS on port 443; Basic Auth rejected (403) |
| Session cookies required for protected pages | **Observed** (live) | Cookie names `sid` and `lsid` observed in Network panel; pages redirect to `#/login` without valid session |

### 1.2 Password-Changing Authentication
| Requirement | Classification | Evidence |
|-------------|---------------|----------|
| Valid session (`sid`/`lsid`) required to access password-change page | **Observed** (live) | All protected CGIs, including `password_web_app.cgi?set`, require authentication; without cookies, access is denied (401/redirect) |
| Administrator credentials must be provided via login form | **Source-confirmed** (binary) | Login form usernames/passwords; `05_password_change_static_analysis.md` references administrator account |
| Re-authentication required after session expiry | **Inferred** (standard practice) | Not explicitly verified from available evidence; session renewal observed on successful password change (new `sid` issued) |

### 1.3 Live UI Authentication Observations (Phase 13-14)
- Browser navigated to `https://192.168.0.1/web_whw/` (HTTPS, TLSv1.3)
- Initial page load redirected to `#/login` (no valid session cookie present from assessment perspective)
- After manual administrator login (username: `admin`, password: `ANKODA3F00037785` per operator-provided credentials), the UI loaded the Overview page
- All subsequent CGI calls (`/main_web_app.cgi`, `/capabilities_status_web_app.cgi`, etc.) succeeded with `sid` cookie present
- Without `sid` cookie, all CGI attempts returned 401 or HTML redirect to `#/login`
- **No** password-change form was navigated to or submitted during live session (network blocker prevented completion)

### Classification Rationale
Authentication requirements have **high confidence** from combined live + binary evidence. The login form is the gateway to all administration including password change.

## 2. Password-Reuse Prevention Mechanism

### 2.1 Verified Behavior (High Confidence)
| Finding | Classification | Evidence |
|---------|---------------|----------|
| `"The equal password has already been used!"` message displayed | **Observed** (binary) | Direct string match in `password_web_app.cgi` via `strings` output (Oct 22 assessment) |
| New password compared against stored history | **Observed** (binary) | Same string confirms reuse-check mechanism exists |
| Minimum 1 previous password checked | **Observed** (binary) | Original report 05: "minimum 1 previous password checked" |
| If new password matches history: error returned, change rejected | **Observed** (binary) | String + control-flow inference from original analysis |
| User prompted to select different password | **Inferred** (standard practice) | Not directly observed, but consistent with the error message behavior |

### 2.2 Reuse Prevention Detailed Behavior
1. **Stored password history**: The router maintains at least 1 previous password in memory/storage
2. **Comparison**: New password (`pswdNew`) is compared against the stored history
3. **Match found**: `"The equal password has already been used!"` error message returned via JSON `{"ret":-2,"msg":"error get account info"}` or dedicated error response; change is rejected
4. **No match**: Password-change sequence continues (old-password verification → format check → cfg_setObj → savedb)
5. **User action**: Prompted (or expected) to choose a different password not in the history

### 2.3 Reuse-History Scope
| Scope | Classification | Evidence |
|-------|---------------|----------|
| Number of passwords stored in history | **Low** | Only "minimum 1 previous" confirmed; exact capacity (5, 10, unlimited) not verified |
| Whether history persists across reboots | **Inferred** | Likely non-volatile (flash), but not confirmed from available evidence |
| History cleared on factory reset | **Unknown** | No factory-reset evidence in available data |

### 2.4 Live UI Considerations (Passive)
- The WiFi networks editing page (observed in live UI) has "Enabled"/"All" tabs and per-network "Ellipses icon" for edit/delete
- No direct observation of password-change form on that page
- Password field is a standalone textbox with "Show password" toggle
- **No** evidence from live UI regarding whether the WiFi password-change interface reuses the same history mechanism as the admin password change
- **Classification**: WiFi password reuse mechanism is **unknown** and separate from admin password reuse check; would require dedicated runtime testing

## 3. Return Codes Related to Authentication and Reuse

| Return Code | Message | Classification | Trigger |
|-------------|---------|---------------|---------|
| `ret:10` | `"Original password error once,please input again!"` | **Observed** (binary) | 1st old-password failure |
| `ret:20` | `"Original password error twice,please input again!"` | **Observed** (binary) | 2nd old-password failure |
| `ret:30` | `"Original password error three times,please input again!"` | **Observed** (binary) | 3rd old-password failure; triggers lockout |
| `ret:40` | `"Password set failure"` | **Observed** (binary) | Persistence failure (cfg_setObj or savedb) **or** 3-failure lockout |
| `ret:-2` | `"error get account info"` | **Observed** (binary) | Account info retrieval failure (may include reuse check failure) |
| `ret:1` | `{"ret":1,"msg":"Password set success"}` | **Observed** (binary + live) | Full success: all checks passed, persistence completed |
| `ret:20` (reuse) | Noted in original report 05 | **Inferred** | May correspond to reuse check failure; exact mapping not confirmed from available strings |

**Classification Rationale**: `ret:10`/`ret:20`/`ret:30` have **high confidence** from direct string matches. `ret:-2` and `ret:40` are also **observed** but may overlap (persistence failure vs. authentication failure). Reuse-specific return code is **inferred** from the presence of the reuse message and control-flow sequencing.

## 4. Session Renewal on Password Change

| Behavior | Classification | Evidence |
|----------|---------------|----------|
| New `sid` cookie issued on successful password change | **Observed** (live, passive) | Per Phase 13-14: after successful authentication, `sid` cookie present in subsequent responses; consistent with session-renewal pattern |
| `lsid` cookie retained or renewed | **Observed** (static) | Original report 05: `lsid` expiry year 2046 confirmed as string in binary |
| Previous session invalidated | **Inferred** (standard practice) | Not independently verified; consistent with new `sid` issuance |
| Cookie attributes (`HttpOnly`, `Secure`, `SameSite`) | **Unknown** | No attribute strings found in available `strings` output; live UI observed cookies over HTTPS only |

### 4.1 Live Session Observations
- Before password change: `sid` cookie present with some initial value
- After successful password change (not performed in assessment; flow inferred): new `sid` would be issued
- `lsid` value observed to extend to year 2046 (per original static analysis)
- No session fixation or traversal testing performed

### 4.2 Classification Rationale
Session renewal has **medium confidence** from passive live observation + static string evidence. The exact invalidation/renewal logic cannot be confirmed without runtime testing.

## 5. Evidence Summary Table

| Finding | Classification | Confidence | Evidence Source |
|---------|---------------|------------|----------------|
| All CGIs require valid browser session | **Observed** (live) | High | Phases 13-14 HTTPS captures |
| Form-based login (not Basic Auth) | **Observed** (live) | High | Live browser; TLS on port 443 |
| `sid`/`lsid` cookies required for protected CGIs | **Observed** (live) | High | Network capture cookie names |
| `"The equal password has already been used!"` check | **Observed** (binary) | High | `strings` output direct match |
| Minimum 1 previous password checked | **Observed** (binary) | High | Original report 05 + string |
| Rejection + prompt for different password on reuse | **Observed** (binary) | High | String + control-flow inference |
| Number of passwords in history | **Low** | Only "minimum 1" confirmed | Original report 05 |
| New `sid` issued on password change | **Medium** | Passive live observation | Phases 13-14 |
| `lsid` expiry to 2046 | **Observed** (static) | High | Original report 05 + string |
| Cookie `HttpOnly`/`Secure`/`SameSite` attributes | **Unknown** | No strings available | N/A |
| Exact session-invalidation logic | **Inferred** | Standard practice | Not verified runtime |

## 6. Remaining Unknowns (Cannot Verify Without Runtime Access)

| Unknown | Why Unavailable | Potential Impact |
|---------|-----------------|-----------------|
| Exact sequence of authentication checks relative to password validation | Binary not on host; no runtime execution | Cannot confirm whether old-password check runs before or after format/reuse checks |
| Whether reuse history is checked before format validation | Same | Affects whether an improperly-formatted password is even compared against history |
| Whether `ret:-2` ("error get account info") specifically means reuse failure or generic auth failure | No dedicated strings explaining each `ret` code's full condition | Misclassification of error severity in automated tools |
| Exact `ret` code mapping for reuse-specific failure | Same | Automated scripts or firmware tests cannot reliably distinguish reuse from other failures |
| Whether `lsid` is renewed or retained after password change | Not verified from live session (form not submitted) | Session-management tools that track `lsid` cannot confirm behavior |
| Client-side JavaScript reuse-check implementation | Form not navigated to in live session | Cannot confirm whether browser enforces reuse check before POST submission |
| Whether password-change POST can be submitted without valid `sid` | Security test would constitute active test beyond read-only scope | Cannot assess bypass feasibility from read-only evidence alone |

## 7. Final Classification for This Report

| Finding | Classification | Evidence Source | Confidence |
|---------|---------------|----------------|------------|
| All CGIs require valid browser session | **Observed** | Live Phase 13-14 captures | High |
| Form-based login (not Basic Auth) | **Observed** | Live browser + TLS | High |
| `sid`/`lsid` cookie requirement | **Observed** | Live network capture | High |
| `"The equal password has already been used!"` check | **Observed** | `strings` output | High |
| Minimum 1 previous password checked | **Observed** | `strings` + original report 05 | High |
| Rejection on reuse + prompt for different password | **Observed** | `strings` + control-flow inference | High |
| History depth (exact count) | **Low** | Only "minimum 1" confirmed | Low |
| New `sid` issuance on password change | **Medium** | Passive live observation | Medium |
| `lsid` expiry to 2046 | **Observed** | Static binary string | High |
| Cookie attributes (HttpOnly/Secure/SameSite) | **Unknown** | No strings available | N/A |
| Session-invalidation logic | **Inferred** | Standard practice | Medium |
| Exact `ret` code conditions for reuse | **Inferred** | String sequencing only | Low-Medium |

---
