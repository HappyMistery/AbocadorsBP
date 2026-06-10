# Security & Refactoring Audit — AbocadorsBP

**Date:** 2026-06-10
**Scope:** `AbocadorsBPAPI` (full backend) and `AbocadorsBPApp` (token storage, API client, WebView map / data flow).
**Method:** Manual source review. No code was changed.

---

## TL;DR

- **1 CRITICAL** account-takeover bug in the API — fix first.
- **2 HIGH** broken-access-control / token-design issues.
- Several **MEDIUM** items (PII exposure, no rate limiting, error leakage).
- **App side is clean**: tokens in SecureStore, sound refresh/401 flow, no XSS surface in the WebView.
- **No secrets committed**: `.env` and `credentials.json` are gitignored and absent from git history.

Priority order to tackle: **#1 → #2 → #3 → MEDIUM → bugs → refactors.**

---

## Vulnerabilities

### 🔴 CRITICAL #1 — Account takeover via leaked password-reset token

**Where:** `AbocadorsBPAPI/app/routers/users.py` — `GET /user/{email}` (lines ~18–37)

`GET /user/{email}` is **unauthenticated** and returns the whole DB row, stripping only the password hash:

```python
user_data = dict(db_user)
user_data.pop("paswd_hash", None)   # only the hash is removed
return {"code": 200, "user": user_data}
```

The `denouncer` row includes `reset_token` and `reset_token_expiry` (`app/db/models.py:13-14`), so both are exposed to anyone.

**Attack chain:**
1. `POST /forgot_password` with the victim's email → a live `reset_token` is written to the DB.
2. `GET /user/victim@example.com` → read `reset_token` straight from the JSON response.
3. `POST /reset_password/` with that token → set a new password and take over the account.

The same leaked token also works against `/delete_account` and `/verify_email_change` (see #3).

**Fix:** Return an explicit allow-list of fields (e.g. `username`, `pfp`, `region`, `municipality`, `is_admin`) instead of dumping the row. Ideally don't even `SELECT` the token columns here.

---

### 🟠 HIGH #2 — `/notify_retire_dumpster` has no authentication

**Where:** `AbocadorsBPAPI/app/routers/dumpsters.py` — `POST /notify_retire_dumpster` (lines ~314–343)

The endpoint has no `Depends(...)`. Any anonymous caller can:
- Flip **any** dumpster to `status='retired'`, silently corrupting the public map.
- Trigger an email to the admin inbox on every call (mail-spam / SMTP-cost vector).

**Fix:** Require `Depends(get_current_user)` at minimum. Better: make this notification-only, and require `require_admin` for the actual status change (the app already distinguishes the admin vs. notify flows in `map.jsx`).

---

### 🟠 HIGH #3 — Token confusion: one column, three purposes

**Where:** `reset_token` column reused across:
- Password reset — `app/routers/auth.py:185`
- Email change — `app/routers/users.py:89`
- Account deletion — `app/routers/pages.py:58`

Every consumer only checks `reset_token == token && not expired`, so a token minted for one action is accepted by the others (e.g. a "change email" link replayed to delete the account).

**Fix:** Give each flow its own token — separate columns, a `purpose` field, or a dedicated short-lived tokens table.

---

### 🟡 MEDIUM #4 — Unauthenticated PII exposure & user enumeration

- `GET /user_info/{id}` (`users.py:117-123`) returns email + username for any integer id → iterate `1..N` to harvest every user's email.
- `GET /user/{email}` is an account-existence oracle (404 vs 200) and leaks `is_admin`, letting an attacker pinpoint admin accounts.
- Per-user stats endpoints (`/user_dumpster_stats`, `/total_reports_stats`, `/user_reports*`) are open. If profiles are intentionally public this is partly by design, but the **email** disclosure is the part to close.

**Fix:** Return only what the public profile screen needs; never expose raw email via id lookup.

---

### 🟡 MEDIUM #5 — No rate limiting anywhere

Nothing throttles:
- `/login` — password brute-force (bcrypt verify is also CPU cost per attempt).
- `/register` and `/forgot_password` — both email an attacker-supplied address → email-bombing a third party + burning the Gmail SMTP quota.
- `/dumpster/predict` — loads the YOLO model **per call** (see refactor #4), so a few requests can pin the 2-vCPU box.

**Fix:** Add `slowapi` or an Nginx `limit_req` zone on these routes.

---

### 🟡 MEDIUM #6 — Internal error details returned to clients

`str(e)` is returned in error bodies:
- `register` / `register_dumpster` — `auth.py:65`, `dumpsters.py:179`
- `edit_user` — `users.py:56`

Leaks stack/driver internals. **Fix:** log server-side, return a generic message.

---

### 🟢 LOW

- **Dependencies:** `requirements.txt` is unpinned (only `bcrypt<4.0.0`). `python-jose` has known advisories (algorithm-confusion / DoS). Pin versions; consider migrating JWT to `PyJWT`. The file is also saved as UTF-16 — re-save as UTF-8.
- **Debug prints in prod:** `print(db_user.email)` (`auth_deps.py:30`) and `print(db_user)` (`auth.py:222`) write PII to logs on every authenticated request.
- **`credentials.json`** stores the Android keystore password in plaintext (`AbocadorsBPApp/credentials.json`). Correctly gitignored and not in history, but treat it (and the `.jks`) as a real secret in backups.

---

## Functional bugs (found during review)

1. **`verify_email_change` crashes for users with no profile picture.**
   `app/routers/pages.py:275-292` — `new_pfp_rel` is only assigned inside `if old_pfp_path:` but is always referenced in the `UPDATE`. A user without a pfp gets a `NameError` → 500 and can never change their email. Guaranteed break.

2. **`register_dumpster` name-sequence race.**
   `app/routers/dumpsters.py:148-158` — next `ABPxxx#####` is `max + 1`; two concurrent reports in the same municipality collide on `UNIQUE(name)` → 500. Use a per-municipality sequence/counter or retry-on-conflict.

3. **`accept_dumpster` doesn't validate waste IDs.**
   `app/routers/dumpsters.py:203-220` — unlike `update_dumpster` (validates 0–7), an out-of-range id raises a raw FK error.

---

## Refactoring proposals (highest leverage first)

1. **Make the API authenticated-by-default.**
   Findings #1, #2, #3 all stem from "forgot to add `Depends`." Today routes are public unless they opt in. Invert it: a router-level dependency that requires auth unless the path is on an explicit public allow-list. Structurally prevents the whole class of bug.

2. **Extract the repeated "read image file → base64" block.**
   The same `os.path.join(__file__, "..", path)` → `abspath` → `open` → `b64encode` appears in 5+ places (`users.py:27-32` & `:305-310`, `dumpsters.py:51-59` & `:110-115`). One `image_to_base64(rel_path)` helper that confines paths to the images root removes duplication and centralizes path-traversal defense.

3. **Extract "fetch user by email or 404."**
   Copy-pasted in ~7 endpoints across `users.py`. A `get_user_or_404(email)` dependency collapses them and makes the field-whitelisting fix (#1) apply in one spot.

4. **Load the YOLO model once.**
   `app/services/yolo.py:21` does `YOLO(MODEL_PATH)` on every `/predict` call. Load at startup (module singleton or lifespan). This is both the DoS mitigation and a large latency win.

5. **Adopt DB migrations (Alembic).**
   Schema is hand-maintained in `createDB.sql` with no tooling; migrations make the auth-column refactor (#3) safe against the live DB.

6. **Standardize the response/error envelope.**
   Mix of `{"code": 200, ...}` bodies and `HTTPException` errors — plus endpoints returning `{"code": 404}` with HTTP 200 (`dumpsters.py:117`) — makes the client guess. Pick one convention.

**App-side minor:** `utils/validation.js` only checks password length ≥ 8 and doesn't mirror the server's reserved-username rule. Fine since the server is authoritative — just be aware they're not mirrored.

---

## What was reviewed and found clean

- **App token storage** (`utils/tokenStorage.js`): access + refresh tokens in `expo-secure-store`.
- **App API client** (`utils/apiClient.js`): single-flight 401→`/refresh` retry, auto-logout on refresh failure, preserves `has_seen_*` flags.
- **WebView map** (`app/(tabs)/map.jsx`): only injects numbers, status-derived CSS classes, and base64 image data into `innerHTML`; user text (name/description/region) is rendered natively (React Native `Text`/`TextInput`) and HTML-escaped server-side → no XSS path.
- **Secrets hygiene**: `.env`, `credentials.json`, `*.jks` not tracked and not in git history.
- **Auth primitives** (`app/core/auth_jwt.py`): bcrypt via passlib, opaque refresh tokens stored as HMAC-SHA256(pepper, token), rotation + max-3-per-user eviction — sound design.

---

## Suggested first PR (small, low-risk, no response-shape changes)

1. `/user/{email}`: replace `dict(db_user)` dump with an explicit field allow-list (fixes #1).
2. `/notify_retire_dumpster`: add `Depends(get_current_user)` (fixes #2).

Both are tiny diffs that don't alter the response shapes the deployed (and older installed) app versions depend on.
