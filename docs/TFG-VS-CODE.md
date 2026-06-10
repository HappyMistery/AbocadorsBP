# TFG document vs. actual code

The thesis (`Documentació TFG/TFG_TelloViñas_Jaume.pdf`, 155 pp., Catalan) describes the
system as designed/deployed around mid-2025. The project kept evolving after
publication. **When the PDF and the code disagree, the code wins.** This file lists the
known divergences so outdated decisions don't leak back into documentation or reviews.

## Authentication & sessions

| TFG says | Code does now |
|---|---|
| No JWT. `/login` validates credentials and returns user data; session = presence of `@user_email` in AsyncStorage. | Full JWT scheme: `/login` returns `access_token` (HS256, 30 min) + rotating opaque `refresh_token` (90 days, HMAC-SHA256 + pepper, max 3 per user). New endpoints `/refresh`, `/logout`; new table `denouncer_refresh_token`. App stores tokens in **expo-secure-store** and auto-refreshes on 401 (`utils/apiClient.js`). |
| Logout = `AsyncStorage.clear()`. | Logout revokes the refresh token server-side and clears local data **except** `has_seen_*` tutorial flags. |
| Protected endpoints not specified; predict/register open. | `get_current_user` / `require_admin` dependencies protect registering, predicting, profile editing and all moderation endpoints. |

## Map / frontend

| TFG says | Code does now |
|---|---|
| Leaflet + MarkerCluster (CDN) in a WebView; OSM/CartoDB/Esri tile layers via `L.tileLayer`. | **MapLibre GL JS** (CDN) in the WebView, in both `map.jsx` and `ManualLocationModal.js`. Variables are still named `leafletHtml` — legacy naming only. CARTO tiles with proper attribution. |
| Direct `fetch("https://abocadorsbp.com:443/...")` calls scattered in screens. | Centralized API layer: `utils/apiWrapper.js` (`apiUrl` from `EXPO_PUBLIC_API_URL`) + `utils/apiClient.js` (`apiFetch`) + domain wrappers in `utils/api/`. |
| Expo described generically (thesis-era SDK). | Expo SDK 54, RN 0.81.5, React 19.1, expo-router v6, New Architecture enabled. Onboarding tutorials via react-native-copilot were added post-thesis. |
| — (not in thesis) | Email-change flow (`/request_email_change`, `/verify_email_change`) and account-deletion flow (`/request_delete_account`, `/delete_account`) exist in app + API. |

Note: `AbocadorsBPApp/README.md` is itself partially stale (says Expo 53 / RN 0.79 /
Router v5 / Leaflet / `utils/api.js`). Trust `package.json` and the code.

## Backend structure & API

| TFG says | Code does now |
|---|---|
| Flat structure: `app/main.py` (everything) + `app/predict_yolo.py` + `app/static/`. | Layered: `core/`, `db/`, `schemas/`, `routers/` (pages, auth, users, dumpsters, stats), `services/` (mailer, yolo), `templates/`. |
| Endpoint `/dumpsters` GET with repeated `ids=` query params; `/dumpsters_paginated` and a route-ordering workaround. | `POST /dumpsters` with JSON body `{"ids": [...]}`. The paginated pending list is `GET /pending_dumpsters` (admin-only). The route-ordering anecdote is obsolete. |
| Swagger/OpenAPI auto-docs as a FastAPI benefit. | Docs are **disabled** in production (`docs_url=None, redoc_url=None, openapi_url=None`). |
| Images ≤ 3.5 MB binary (≈5 MB base64). | Limit is **4 MB binary** / 5,000,000 base64 chars, JPEG magic-bytes enforced. |
| Plain-text emails (`subtype="plain"`). | HTML emails for verification/reset/email-change; plain text only for the admin retire notification. |
| `/forgot_password` errors when email doesn't exist. | Returns a generic 200 regardless (anti user-enumeration). |
| `static/privacy_policy.html` served by FastAPI. | `app/templates/privacy.html` + `welcome.html`; `/privacy` is served statically by **Nginx**; legacy `/privacy_policy` URLs 301-redirect. |
| — | `/dumpster/register`: admins' reports are auto-`accepted`; users can only report as themselves. Reserved usernames blocked (admin/root/system/self/abocadorsbp). |

## Deployment

| TFG says | Code does now |
|---|---|
| Uvicorn binds `0.0.0.0:443` directly with `--ssl-certfile`/`--ssl-keyfile`. | **Nginx** terminates TLS on 80/443 and reverse-proxies to Uvicorn on `127.0.0.1:8000` (systemd unit, 5 workers). |
| Project at `/root/AbocadorsBPAPI`, Python 3.11 venv. | Deployed at `/srv/AbocadorsBPAPI`, Python **3.13.3** (the API README calls it strict). |
| Cron deletes stale `pending_denouncer` hourly (`0 * * * *`). | Cron runs **every minute** (`* * * * *`), same 1-hour staleness condition. |
| `.env` holds DB + mail config only. | `.env` also holds `JWT_SECRET_KEY`, `JWT_ALGORITHM`, `ACCESS_TOKEN_EXPIRE_MINUTES`, `REFRESH_TOKEN_PEPPER`, `REFRESH_TOKEN_EXPIRE_DAYS`, and optional `EXPO_PUBLIC_API_URL`/`PUBLIC_API_URL` for dev. |

## Still accurate in the TFG

These thesis sections remain reliable and are good background reading:

- The **TFG training repo** as a whole (datasets, TACO→YOLO conversion, class
  remapping, labeling/validation tools, Ray Tune setup, training chronology
  train–train26, train25 as production model). It hasn't changed since Aug 2025.
- The **database core schema** (denouncer, pending_denouncer, dumpster,
  rejected_dumpster, waste, dumpster_waste) — only `denouncer_refresh_token` was added.
- The **dumpster naming scheme** (`ABP` + municipality code + 5 digits, `ABPUNK`
  fallback) and `comarques_municipis.json`.
- The 8 waste classes and their IDs.
- The **app's screen map** (tabs map/camera/profile; login/register; edit_profile;
  manage_dumpsters; dynamic profile/user_reports routes), the WebView messaging design
  (`bboxChange`, `dumpClick`, `requestDumpsterImage`, ...), viewport-driven loading with
  in-memory caches, GeoJSON slimming (~58% reduction), AsyncStorage `@user_*` keys,
  compression presets, theme system and color palette.
- Hetzner CX22 / Falkenstein / Ubuntu 24.04 infrastructure, SSH-key-only access,
  firewall policy, PostgreSQL bound to localhost.

## Things the TFG promises that don't exist (yet)

- Web portal at abocadorsbp.com embedding the labeling/validation tools for
  crowdsourced annotation.
- YOLOv12 migration / retraining with user-submitted images.
- PostGIS usage (PostgreSQL chosen partly for it, but plain FLOAT columns are used).
- Stratified dataset split / class rebalancing.
