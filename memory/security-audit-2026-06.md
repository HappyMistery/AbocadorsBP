---
name: security-audit-2026-06
description: Findings from the 2026-06-10 security review of AbocadorsBPAPI/App; lists known-but-unfixed vulns.
metadata:
  type: project
---

On 2026-06-10 a full security/code review of `AbocadorsBPAPI` and `AbocadorsBPApp` was done. Key findings (unfixed as of that date unless noted):

- **CRITICAL — account takeover via reset_token leak.** `GET /user/{email}` (unauthenticated) in `AbocadorsBPAPI/app/routers/users.py` returns `dict(db_user)` popping only `paswd_hash`, so `reset_token`/`reset_token_expiry` are exposed. Attacker triggers `/forgot_password`, polls `/user/{email}`, reads the live reset token, then `/reset_password/` (or `/delete_account`, `/verify_email_change`) to hijack the account. Fix: whitelist returned fields.
- **HIGH — `/notify_retire_dumpster` has no auth dependency** (`routers/dumpsters.py`); anyone can set any dumpster `status='retired'` and trigger admin emails.
- **HIGH — token confusion**: one `denouncer.reset_token` column is reused for password reset, email change, and account deletion — a token issued for one purpose works for the others.
- MEDIUM: unauthenticated PII/enumeration (`/user/{email}`, `/user_info/{id}`, user stats endpoints); no rate limiting (login brute force, register/forgot_password email bombing); internal error strings leaked via `str(e)` in 500s; `python-jose` + unpinned `requirements.txt`; stray `print()` of emails/rows.
- BUG: `verify_email_change` raises `NameError` (uses `new_pfp_rel` undefined) when the user has no pfp.

Secrets are NOT committed (`.env`, `credentials.json` gitignored and absent from history). The app side (SecureStore tokens, `apiClient.js` refresh flow, WebView) reviewed clean.
