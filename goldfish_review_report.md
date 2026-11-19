# Goldfish Annotated Review Report

**Status:** `RESOLVED (Auto-Accepted and Applied)`  
**Target:** `aepl-invoice-automation` (`main.py`, `index.html`, `functions/index.js`, `.github/workflows/deploy.yml`) (`full codebase`)

## Executive Summary & Status
This report aggregates all verified security vulnerabilities identified during the comprehensive security audit and system review of the AEPL SAP Invoice Engine. Each finding has been evaluated and categorized according to the **EGM Protocol** into `ESCALATE`, `PROPOSE FIX`, or `CONTEST`.

- **Total Findings:** 24
- **Escalated (Requires User Infrastructure Knowledge):** 1 (`CLOUD_RUN_VPC_DISCONNECTION`)
- **Proposed Fixes (Ready for Automated Batch Application):** 23
- **Contested:** 0

---

## Escalated Findings (Requires User Confirmation)

### [Issue 1] [Hunter Frontend & Cloud] - Severity: Major
*   **Affected Scope:** [Line 38](.github/workflows/deploy.yml#L38) (`jobs.deploy.steps[3].run`)
*   **Discrepancy Description:** The GitHub Actions deployment workflow passes `--clear-network` to `gcloud run deploy`, actively detaching Cloud Run from the private Direct VPC network (`vpc-aepl-inv-sec`, subnet `sub-aepl-us`). This causes outbound SAP CPI BAPI traffic to route over the public Internet.
*   **Rationale:** README.md (lines 159-163) explicitly requires VPC network egress for private SAP S/4HANA ERP communication.
*   ***AI Suggestion:*** `ESCALATE`
*   ***Justification:*** Whether Cloud Run must attach to `vpc-aepl-inv-sec` depends on whether the SAP CPI tenant has public IP endpoints or is strictly internal to the private VPC subnet.
*   ***Resolution / Action:*** Retained `--clear-network` in `.github/workflows/deploy.yml` to prevent Cloud Run startup probe failures (4m probe timeout) until enterprise network infrastructure activates VPC routes and Private Google Access for `sub-aepl-us`.

---

## Prioritized Findings & AI Suggestions

### [Issue 2] [Hunter Auth & Session] - Severity: Blocker
*   **Affected Scope:** [Lines 194-265, 1145-1175](main.py#L194-L265) (`PREDEFINED_USERS` and `get_user`)
*   **Discrepancy Description:** `PREDEFINED_USERS` hardcodes plaintext passwords (`AuroAdmin@2026!` and `AuroUser@2026!`). If Firestore is uninitialized or unreachable, `get_user` returns `_USERS_MEM` seeded with these default credentials.
*   **Rationale:** Static default administrative passwords enable immediate unauthorized account takeover.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Plaintext credentials must be completely removed from source code. Users must be authenticated exclusively against Firestore password hashes or environment-seeded credentials.
*   ***Proposed Edit:***
    *   Location: `main.py:194-265` and `main.py:1145-1175`
    *   Change: Remove `default_pwd` from `PREDEFINED_USERS`. In `get_user`, never return a hardcoded password hash if Firestore is unavailable; fail closed.

---

### [Issue 3] [Hunter Auth & Session] - Severity: Blocker
*   **Affected Scope:** [Line 1799](main.py#L1799) (`SESSION_SECRET`)
*   **Discrepancy Description:** If `SESSION_SECRET` is not set in the environment, the application falls back to static string `'aepl-invoice-secure-session-secret-2026-key'`.
*   **Rationale:** An attacker with knowledge of the open-source repository can forge HMAC-SHA256 tokens offline with `'is_admin': True`, bypassing all authentication gates.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Enforce mandatory `SESSION_SECRET` at runtime. Fail fast or generate an ephemeral random 32-byte secret per instance if unset.
*   ***Proposed Edit:***
    *   Location: `main.py:1799`
    *   Change: `SESSION_SECRET = os.getenv("SESSION_SECRET") or secrets.token_hex(32)` and log a high-priority security warning if ephemeral fallback is used.

---

### [Issue 4] [Hunter Auth & Session] - Severity: Blocker
*   **Affected Scope:** [Lines 3449-3464](main.py#L3449-L3464) (`forgot_password`)
*   **Discrepancy Description:** `/api/auth/forgot-password` accepts an unvalidated `redirect_origin` from the client request payload and interpolates it into the password reset URL sent to users and corporate administrators.
*   **Rationale:** Attackers can poison password recovery emails to harvest one-time reset tokens and take over accounts.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Whitelist allowed redirect origins against trusted corporate domains (`https://prj-aepl-ai-inv-auto-dev.web.app`, `https://prj-aepl-ai-inv-auto-dev.firebaseapp.com`, localhost).
*   ***Proposed Edit:***
    *   Location: `main.py:3449-3464`
    *   Change: Validate `payload.redirect_origin` against an approved whitelist and default to the canonical production URL if untrusted.

---

### [Issue 5] [Hunter Frontend & Session] - Severity: Major
*   **Affected Scope:** [Lines 1968-1977, 2609-2620, 4200-4220](index.html#L1968-L1977) (`isSessionIdleExpired`, `verifyActiveSession`, Firebase Persistence)
*   **Discrepancy Description:** Users remain logged in across days in Edge browser because: (1) `isSessionIdleExpired` returns `false` when `last_activity_at` is missing in `sessionStorage`; (2) `allowOfflineRestore` grants access without backend verification; (3) Firebase Auth persists across browser restarts in IndexedDB.
*   **Rationale:** Violates corporate security policies requiring sessions to expire after 2 hours of inactivity or 8 hours total duration.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Enforce: (a) missing `last_activity_at` treated as expired; (b) remove `allowOfflineRestore` auto-authentication; (c) enforce `setPersistence(auth, inMemoryPersistence)` on Firebase Auth; (d) purge stale credentials on startup.
*   ***Proposed Edit:***
    *   Location: `index.html:1968-1977`, `index.html:2609-2620`, `index.html:4205`
    *   Change: Return `true` if `raw === null` in `isSessionIdleExpired`; require 200 OK from `/api/auth/me` to render portal; set in-memory persistence for Firebase.

---

### [Issue 6] [Hunter Ingestion & AI] - Severity: Major
*   **Affected Scope:** [Lines 2006-2008](main.py#L2006-L2008) (`verify_with_model_armor`)
*   **Discrepancy Description:** `verify_with_model_armor` catches broad `Exception` and returns `(False, "")`, failing open.
*   **Rationale:** When Model Armor experiences quota exhaustion, timeout, or configuration errors, unverified LLM responses are sent directly to SAP CPI and committed as corporate liabilities.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Fail closed in production. Route uninspected invoices to `HITL_REQUIRED` quarantine when Model Armor screening fails.
*   ***Proposed Edit:***
    *   Location: `main.py:2006-2008`
    *   Change: Return `(True, f"Model Armor inspection failure: {e}")` to trigger HITL routing.

---

### [Issue 7] [Hunter SAP HITL Alerts] - Severity: Major
*   **Affected Scope:** [Line 3943](main.py#L3943) (`resolve_hitl_invoice`)
*   **Discrepancy Description:** Submitter tenant check evaluates to `False` when `submitted_by` is empty/null, allowing non-admin users to resolve and park other users' invoices in SAP.
*   **Rationale:** Bypasses tenant boundary and authorization controls.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Enforce that non-admin callers can only resolve invoices where `submitted_by` strictly equals the caller's verified email.
*   ***Proposed Edit:***
    *   Location: `main.py:3943`
    *   Change: `if not is_admin and (not submitted_by or submitted_by != user_email.lower()): raise HTTPException(status_code=403, detail="Unauthorized")`

---

### [Issue 8] [Hunter SAP HITL Alerts] - Severity: Major
*   **Affected Scope:** [Line 3970](main.py#L3970) (`resolve_hitl_invoice`)
*   **Discrepancy Description:** `resolve_hitl_invoice` explicitly allows `IRRELEVANT_DOC_PENDING` invoices to be resolved and posted into SAP without validating company code.
*   **Rationale:** Third-party or personal bills flagged as irrelevant can be booked as financial liabilities in SAP.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Disallow posting `IRRELEVANT_DOC_PENDING` documents to SAP; require re-classification and administrative approval.
*   ***Proposed Edit:***
    *   Location: `main.py:3970`
    *   Change: Remove `IRRELEVANT_DOC_PENDING` from auto-resolvable status list.

---

### [Issue 9] [Hunter SAP HITL Alerts] - Severity: Major
*   **Affected Scope:** [Lines 1280-1295](main.py#L1280-L1295) (`find_duplicate_invoice`)
*   **Discrepancy Description:** Queries Firestore with `.limit(50)` and filters client-side in Python, ignoring in-flight invoices.
*   **Rationale:** Once 50 invoices exist, duplicates are never detected, allowing duplicate vendor disbursements.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Query Firestore by `RefDocNo` and `CompCode` directly without the 50-item limit, checking both `PARKED_SUCCESS` and in-flight statuses.
*   ***Proposed Edit:***
    *   Location: `main.py:1280-1295`
    *   Change: Construct targeted Firestore query matching `ref_doc_no` and `comp_code`.

---

### [Issue 10] [Hunter SAP HITL Alerts] - Severity: Major
*   **Affected Scope:** [Lines 2150-2272](main.py#L2150-L2272) (`submit_to_sap_cpi`)
*   **Discrepancy Description:** If SAP CPI credentials are unset, the system silently generates synthetic BELNR numbers and records status `PARKED_SUCCESS` in Firestore.
*   **Rationale:** Causes phantom invoices where the portal shows invoices successfully parked in SAP when no call was ever made.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Reject mock simulation in production. Fail with explicit 503 error if SAP credentials are not configured.
*   ***Proposed Edit:***
    *   Location: `main.py:2150-2272`
    *   Change: Check environment; if production, abort with error instead of executing mock simulation.

---

### [Issue 11] [Hunter Ingestion & AI] - Severity: Major
*   **Affected Scope:** [Lines 2016-2054](main.py#L2016-L2054) (`optimize_image_768px`)
*   **Discrepancy Description:** Pillow `Image.open` does not restrict `MAX_IMAGE_PIXELS`, decompressing raw images and creating uncompressed full-scale PDF copies multiple times in memory.
*   **Rationale:** Decompression bomb DoS can exhaust container memory and crash Cloud Run instances.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Set `Image.MAX_IMAGE_PIXELS = 25_000_000` and process images in a single stream.
*   ***Proposed Edit:***
    *   Location: `main.py:2016-2054`
    *   Change: Add `Image.MAX_IMAGE_PIXELS = 25000000` and catch `DecompressionBombError`.

---

### [Issue 12] [Hunter SAP HITL Alerts] - Severity: Major
*   **Affected Scope:** [Lines 23-47](functions/index.js#L23-L47) (`sendFailureEmail`)
*   **Discrepancy Description:** Cloud Function forwards emails to `mailData.to[0]` via Brevo REST API without domain whitelisting.
*   **Rationale:** Acts as an open email relay for arbitrary external addresses written to Firestore `mail` collection.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Validate recipient domain against `@aurogroup.com` or configured allowed domains.
*   ***Proposed Edit:***
    *   Location: `functions/index.js:24-30`
    *   Change: Check `recipient.endsWith("@aurogroup.com")`; if invalid, fall back to default corporate admin email.

---

### [Issue 13] [Hunter Frontend & Cloud] - Severity: Major
*   **Affected Scope:** [Lines 3197-3697](index.html#L3197-L3697) (`renderDocResultCard`, `renderMISInvoicesTable`, `renderMISAuditLogs`)
*   **Discrepancy Description:** Concatenates unescaped invoice numbers, vendor names, and item texts directly into `innerHTML`.
*   **Rationale:** Enables Stored and DOM XSS under permissive CSP.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Wrap all dynamic variables in `escapeHtml(...)` before template interpolation.
*   ***Proposed Edit:***
    *   Location: `index.html:3197-3697`
    *   Change: Pass `header.RefDocNo`, `header.CompName`, `it.ItemText`, `inv.filename`, etc. through `escapeHtml()`.

---

### [Issue 14] [Hunter Auth & Session] - Severity: Major
*   **Affected Scope:** [Lines 3658-3679](main.py#L3658-L3679) (`record_login`)
*   **Discrepancy Description:** Public unauthenticated endpoint writes untrusted data into `auth_events` and chained `audit_logs`.
*   **Rationale:** Allows external attackers to corrupt the SHA-256 audit log chain and poll for valid admin accounts.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Require authenticated session token and verify caller identity.
*   ***Proposed Edit:***
    *   Location: `main.py:3658-3679`
    *   Change: Add `verify_corporate_user(authorization, x_firebase_token)`.

---

### [Issue 15] [Hunter Auth & Session] - Severity: Major
*   **Affected Scope:** [Lines 272-308](main.py#L272-L308) (`record_failed_login`)
*   **Discrepancy Description:** In-memory lockout cache locks accounts across all IPs for 30 minutes on 5 failed attempts.
*   **Rationale:** Unauthenticated attackers can lock corporate executives out of their accounts indefinitely.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Key lockout by `(identifier, client_ip)` pair rather than identifier alone.
*   ***Proposed Edit:***
    *   Location: `main.py:278-308`
    *   Change: Lock the specific client IP and identifier pair rather than freezing the user globally.

---

### [Issue 16] [Hunter Frontend & Cloud] - Severity: Nit
*   **Affected Scope:** [Lines 126-127](main.py#L126-L127) (`add_security_headers`)
*   **Discrepancy Description:** Content-Security-Policy contains `'unsafe-inline'` for `script-src`.
*   **Rationale:** Weakens defenses against script injection.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Remove `'unsafe-inline'` from script-src or add hash/nonce protections.
*   ***Proposed Edit:***
    *   Location: `main.py:126-127`
    *   Change: Tighten CSP header directives.

---

### [Issue 17] [Hunter Ingestion & AI] - Severity: Nit
*   **Affected Scope:** [Line 2747](main.py#L2747) (`extract_invoice_with_gemini`)
*   **Discrepancy Description:** `abs(candidate_amt - (total_gross - (total_gross - candidate_amt))) < 2.0` is a tautology (evaluates to `0.0 < 2.0`, always true).
*   **Rationale:** Erroneously overwrites legitimate TCS tax amounts.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Correct formula to check actual balance: `abs((subtotal + gst_total + candidate_amt) - total_gross) < 2.0`.
*   ***Proposed Edit:***
    *   Location: `main.py:2747`
    *   Change: Replace tautological equation with the balance reconciliation formula.

---

### [Issue 18] [Hunter SAP HITL Alerts] - Severity: Nit
*   **Affected Scope:** [Lines 530-554](main.py#L530-L554) (`send_failure_alert`)
*   **Discrepancy Description:** User-controlled filenames and error details interpolated into HTML email templates without escaping.
*   **Rationale:** Permits HTML injection into administrative alert emails.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Escape all parameters with `html.escape()` before formatting email body.
*   ***Proposed Edit:***
    *   Location: `main.py:530-554`
    *   Change: Use `html.escape(filename)` and `html.escape(error_detail)`.

---

### [Issue 19] [Hunter SAP HITL Alerts] - Severity: Nit
*   **Affected Scope:** [Line 4047](main.py#L4047) (`resolve_hitl_invoice`)
*   **Discrepancy Description:** Replaces missing invoice attachment with a 45-byte dummy PDF (`%PDF-1.4\n%AEPL HITL Synthesized Document`).
*   **Rationale:** Legal document integrity is lost when multi-instance Cloud Run loses in-memory cache.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Require persistent attachment storage in Firestore/Cloud Storage; reject posting if attachment is lost.
*   ***Proposed Edit:***
    *   Location: `main.py:4047`
    *   Change: Reject resolution if original attachment cannot be recovered.

---

### [Issue 20] [Hunter Ingestion & AI] - Severity: Nit
*   **Affected Scope:** [Lines 3681-3700](main.py#L3681-L3700) (`park_batch_invoices`)
*   **Discrepancy Description:** Single rate limit check for entire batch upload allows uploading hundreds of files in one HTTP request.
*   **Rationale:** Bypasses per-file rate limiting.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Enforce maximum batch size limit (e.g. 10 files) and charge rate limit quota per file.
*   ***Proposed Edit:***
    *   Location: `main.py:3681-3700`
    *   Change: Cap batch upload at 10 files per request.

---

### [Issue 21] [Hunter Ingestion & AI] - Severity: Nit
*   **Affected Scope:** [Line 151](main.py#L151) (`validate_file_magic_bytes`)
*   **Discrepancy Description:** Logical `or` accepts any file extension if content begins with `%PDF`.
*   **Rationale:** Disguised executables with valid PDF headers can pass validation.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Require both allowed extension AND matching magic bytes.
*   ***Proposed Edit:***
    *   Location: `main.py:151`
    *   Change: Replace `or` with `and`.

---

### [Issue 22] [Hunter SAP HITL Alerts] - Severity: Nit
*   **Affected Scope:** [Line 1281](main.py#L1281) (`find_duplicate_invoice`)
*   **Discrepancy Description:** Re-uploading an invoice previously in `SAP_REJECTED` re-submits to SAP.
*   **Rationale:** Bypasses SAP rejection finality guardrails.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Prevent automatic resubmission of rejected invoices without administrative override.
*   ***Proposed Edit:***
    *   Location: `main.py:1281`
    *   Change: Check for `SAP_REJECTED` status and block duplicate resubmission.

---

### [Issue 23] [Hunter SAP HITL Alerts] - Severity: Nit
*   **Affected Scope:** [Lines 2090-2130](main.py#L2090-L2130) (`validate_sap_schema`)
*   **Discrepancy Description:** Schema validation omits RefDocNo check and line item validation.
*   **Rationale:** Permits invoices without vendor invoice references to proceed to SAP CPI submission.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Add RefDocNo and line item validation to `validate_sap_schema`.
*   ***Proposed Edit:***
    *   Location: `main.py:2090-2130`
    *   Change: Validate RefDocNo and ItemData list in schema validator.

---

### [Issue 24] [Hunter SAP HITL Alerts] - Severity: Nit
*   **Affected Scope:** [Line 2093](main.py#L2093) (`parse_sap_cpi_response`)
*   **Discrepancy Description:** Uses standard library `xml.etree.ElementTree` to parse SAP CPI response text.
*   **Rationale:** Vulnerable to entity expansion / XXE if untrusted XML is processed.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** Use `defusedxml.ElementTree` or disable entity resolution.
*   ***Proposed Edit:***
    *   Location: `main.py:2093`
    *   Change: Import and use `defusedxml.ElementTree` where available.

---

### [Issue 25] [RBAC & Access Control] - Severity: Blocker
*   **Affected Scope:** [index.html:1170, index.html:1765, index.html:2240, index.html:3925](index.html#L1170), [main.py:4370-4490](main.py#L4370-L4490)
*   **Discrepancy Description:** Non-admin corporate users (such as Someswara Rao Konatham, role `user`, Accounts Payable Officer) have visibility and access to the "Admin MIS & Monitoring" dashboard tab (`#tabBtnMIS`) and can execute requests to `/api/mis/dashboard`, `/api/mis/invoices`, `/api/mis/audit-logs`, and `/api/mis/incident-alerts` without receiving HTTP 403 Forbidden.
*   **Rationale:** Executive processing telemetry, organization-wide invoice ledgers, system audit logs, and security incident alerts must be strictly restricted to Administrators (`ADMIN_EMAILS`) under enterprise RBAC policies.
*   ***AI Suggestion:*** `PROPOSE FIX`
*   ***Justification:*** 
    1. Frontend: Hide `tabBtnMIS` by default and display only when `isCurrentUserAdmin()` is True. In `switchTab()`, intercept attempts by non-admins to open `mis`, display access denied warning, and revert to `upload`. In `loadMISDashboard()`, abort if not admin.
    2. Backend: Enforce strict `if not is_admin_user(user_email): raise HTTPException(status_code=403, detail="Forbidden: Admin privileges required...")` on `/api/mis/dashboard`, `/api/mis/invoices`, `/api/mis/audit-logs`, and `/api/mis/incident-alerts`.
    3. Multi-User Tenant Isolation: Fix `discard_irrelevant_invoice` and `get_invoice_cpi_response` to fail closed if `submitted_by` is missing.
*   ***Proposed Edit:***
    *   Location: `index.html:1170, 1765, 1835, 1910, 2245, 3925` and `main.py:54, 164, 4300, 4342, 4370-4490`
    *   Change: Applied strict RBAC gating across UI tabs, client routing, and REST endpoints. Status: Resolved.

---
