# Security & Refactoring Audit — AbocadorsBP — open items

**Date:** 2026-06-10 (audit) · **Updated:** 2026-06-12 (trimmed to open items only)
**Scope:** `AbocadorsBPAPI` (full backend) and `AbocadorsBPApp` (token storage, API client, WebView map / data flow).

All CRITICAL/HIGH findings, MEDIUM #4 (PII / user enumeration) and #6 (error leakage),
and the LOW dependency items were **fixed in code on 2026-06-11** and are no longer
tracked here. The practices they introduced are documented as conventions/architecture:
minimal-SELECT, the id-keyed PII-free public surface, self-only email-keyed endpoints,
and no-`str(e)`-in-responses in [CONVENTIONS.md](CONVENTIONS.md); per-flow tokens,
`db/queries.py`/`services/images.py` helpers, the YOLO singleton and the
`dumpster_name_counter` table in [ARCHITECTURE.md](ARCHITECTURE.md).

Refactoring proposals **#1 (authenticated-by-default)** and **#6 (response/error
envelope)** were **applied on 2026-06-12**: a global `require_auth_by_default`
dependency now rejects any route not on the explicit `PUBLIC_PATHS` allow-list
(validated against registered routes at startup), and the three soft-error
`{"code": 400/404}`-with-HTTP-200 responses in `dumpsters.py` became real
`HTTPException`s (backward compatible — app wrappers already treat anything that is
not `res.ok` + `code: 200` as a failure). Both are documented in
[CONVENTIONS.md](CONVENTIONS.md) / [ARCHITECTURE.md](ARCHITECTURE.md). They are
code-only changes in `AbocadorsBPAPI` (no DB migration, no app release needed) and
ride the same pending deployment below.

The **app-side minor note** (validation mirroring) was also resolved on 2026-06-12:
`utils/validation.js` now mirrors the server's reserved-username rules via
`isReservedUsername()` (used in `edit_profile.jsx`); password-wise the server only
enforces length ≥ 8, which the app already mirrored. The server stays authoritative.

---

## Pending deployment (code done, not yet shipped)

The 2026-06-11 fixes are **uncommitted in both repos**, pending manual testing. To ship:

1. Run `AbocadorsBPAPI/migration_2026-06-11_security_audit.sql` on the **prod** Hetzner DB
   (applied to dev only so far; idempotent; migrate-first is safe — old code ignores the
   new columns/table).
2. `pip install -r requirements.txt` in the prod venv (python-jose → PyJWT; pinned versions).
3. Deploy API, then release the app (id-keyed profile navigation).

⚠️ Once this API reaches prod, **old installed app versions** keep the map, markers and
their own profile, but can no longer open *other* users' profiles — that hole is the
MEDIUM #4 fix; it cannot be kept open and fixed at once.

---

## Open vulnerabilities

### 🟡 MEDIUM #5 — No rate limiting anywhere

Nothing throttles:
- `/login` — password brute-force (bcrypt verify is also CPU cost per attempt).
- `/register` and `/forgot_password` (and `/request_email_change`, `/request_delete_account`)
  — all email an attacker-supplied address → email-bombing a third party + burning the Gmail SMTP quota.
- `/dumpster/predict` — YOLO inference on the 2-vCPU box (the model is now loaded once, but a
  burst of calls can still pin CPU).

**Status:** step-by-step plan written in [`docs/PLAN-RATE-LIMITING.md`](PLAN-RATE-LIMITING.md)
(Nginx `limit_req` recommended over `slowapi`, since prod runs 5 Uvicorn workers). Not yet deployed.

### 🟢 LOW — `credentials.json` (operational, no code change possible)

`AbocadorsBPApp/credentials.json` stores the Android keystore password in plaintext.
Correctly gitignored and not in history, but treat it (and the `.jks`) as a real secret in
backups — this is how `eas build --local` credentials work. Mitigation options: exclude it
from any backup that isn't itself encrypted, or switch to EAS-managed remote credentials
(`eas credentials`, stores keystore + passwords on Expo's servers) and delete the local copy.

---

## Open refactoring proposals

### 5. Adopt DB migrations (Alembic)

Schema is hand-maintained in `createDB.sql`, now supplemented by ad-hoc
`migration_*.sql` files. Proper migration tooling would make future schema changes safe and
repeatable against the live DB.
