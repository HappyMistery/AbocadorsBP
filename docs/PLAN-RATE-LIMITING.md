# Plan — Rate limiting for the AbocadorsBP API (audit MEDIUM #5)

**Status:** plan only, nothing implemented yet.
**Goal:** throttle the abuse-prone endpoints flagged in `SECURITY-AUDIT-2026-06.md`:
`/login` (brute force), `/register`, `/forgot_password`, `/request_email_change`,
`/request_delete_account` (email-bombing / SMTP quota), and `/dumpster/predict` (CPU).

## Recommendation: Nginx `limit_req`, not slowapi

The prod box runs **Nginx → Uvicorn with 5 workers**. `slowapi`'s default storage is
in-process memory, so each Uvicorn worker would keep its own counters and every limit
would effectively be ×5 (fixing that requires adding Redis). Nginx already sits in
front, sees the real client IP, shares one counter store, and needs **zero code or
dependency changes**. Use Nginx; keep slowapi as a later defense-in-depth option.

## Step-by-step

### 1. Define the zones (in the `http {}` block, e.g. `/etc/nginx/nginx.conf` or a `conf.d/ratelimit.conf` include)

```nginx
# 1 MB ≈ 16k distinct IPs tracked per zone
limit_req_zone $binary_remote_addr zone=auth:1m      rate=10r/m;  # login
limit_req_zone $binary_remote_addr zone=mailers:1m   rate=3r/m;   # endpoints that send email
limit_req_zone $binary_remote_addr zone=predict:1m   rate=6r/m;   # YOLO inference
limit_req_status 429;
```

### 2. Apply them per-route (in the `server {}` block for abocadorsbp.com, *before* the catch-all `location /`)

Nginx picks the most specific `location`, so these must replicate the same
`proxy_pass`/headers as the existing catch-all. Factor the proxy directives into an
include (e.g. `/etc/nginx/snippets/abp_proxy.conf`) to avoid drift:

```nginx
location = /login {
    limit_req zone=auth burst=5 nodelay;
    include snippets/abp_proxy.conf;
}
location ~ ^/(register|forgot_password|request_email_change|request_delete_account)$ {
    limit_req zone=mailers burst=3 nodelay;
    include snippets/abp_proxy.conf;
}
location = /dumpster/predict {
    limit_req zone=predict burst=2 nodelay;
    include snippets/abp_proxy.conf;
}
```

`burst + nodelay` lets a legitimate user retry a few times instantly, then rejects
with 429 instead of queueing.

### 3. Pick limits deliberately

- `/login` 10/min: a user mistyping a password stays well under it; a brute force
  is capped at ~14k attempts/day/IP (bcrypt cost stops being a DoS vector).
- Mailer endpoints 3/min + burst 3: registration/reset flows send at most one email
  per user action; this caps third-party email-bombing and Gmail quota burn.
- `/dumpster/predict` 6/min + burst 2: photographing a dumpster is a slow human
  action; this keeps the 2-vCPU box responsive. (The model is now loaded once per
  worker, so per-call cost is inference only.)
- Caveat: mobile clients behind CGNAT can share an IP. If 429s show up for real
  users in logs, raise the rates or key the zone on something finer than IP.

### 4. Roll out safely

1. Add the config with generous rates first (e.g. double the table above).
2. `nginx -t` to validate, then `systemctl reload nginx` (reload, not restart — no
   dropped connections).
3. Test from a shell: `for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code}\n" -X POST https://abocadorsbp.com/login -H 'Content-Type: application/json' -d '{"email":"x@x.com","password":"wrongwrong"}'; done`
   → expect the tail of the runs to be `429`.
4. Verify the app handles 429 gracefully (it surfaces a generic error today; that is
   acceptable but worth a friendly message later).

### 5. Monitor, then tighten

- 429s are logged in the Nginx error log (`limiting requests, excess: ...`). Watch
  for a week: `grep "limiting requests" /var/log/nginx/error.log`.
- Tighten to the table above once it's clear real users never hit the limits.

### Out of scope for this plan (possible later hardening)

- slowapi + Redis for per-account (not per-IP) lockout on `/login`.
- `limit_conn` for concurrent connection caps.
- fail2ban on repeated 429/401 offenders.
