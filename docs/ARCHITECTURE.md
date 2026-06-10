# Architecture (verified against code, June 2026)

```
┌─────────────────────┐         HTTPS          ┌──────────────────────────────────────┐
│  AbocadorsBPApp     │  ───────────────────▶  │  Hetzner CX22 (Falkenstein, Ubuntu)  │
│  React Native/Expo  │                        │  Nginx :80/:443 (TLS, Let's Encrypt) │
│  MapLibre WebView   │                        │    └─▶ Uvicorn 127.0.0.1:8000 (x5)   │
│  SecureStore (JWT)  │                        │         └─▶ FastAPI (app.main:app)   │
└─────────────────────┘                        │              ├─ PostgreSQL 16 (local)│
                                               │              ├─ YOLOv11 model.pt     │
                                               │              └─ images/ on disk      │
                                               └──────────────────────────────────────┘
```

## 1. Backend — `AbocadorsBPAPI/`

Layered FastAPI app (see its `README.md`, which is current and detailed):

```
app/
  main.py            # FastAPI instance; Swagger/ReDoc/OpenAPI DISABLED; registers routers
  core/
    config.py        # .env via python-dotenv → constants (DB, mail, JWT, refresh tokens)
    auth_jwt.py      # access JWT (HS256, sub=user id, 30 min) + opaque refresh tokens
    auth_deps.py     # get_current_user (HTTPBearer) / require_admin dependencies
    api.py           # apiUrl() absolute URL builder (PUBLIC_API_URL → prod default)
  db/
    base.py          # SQLAlchemy Base + `databases.Database` async connection, lifespan
    models.py        # table mappings (SQLAlchemy Core style, queried via database.fetch_*)
  schemas/           # Pydantic v2 request models with sanitizing validators
  routers/
    pages.py         # browser HTML: GET /, verify_email, reset_password page,
                     #   request_delete_account, delete_account, verify_email_change
    auth.py          # /register /login /refresh /logout /forgot_password /reset_password/
    users.py         # /user/{email}, /edit_user/, /request_email_change, /user_info/{id},
                     #   /user_dumpster_stats, /total_reports_stats,
                     #   /user_monthly_report_stats, /user_reports, /user_reports_images
    dumpsters.py     # /dumpster_wastes/{id}, /dumpster_stats/{area}, /pending_dumpsters,
                     #   /dumpster_ids_in_bbox, POST /dumpsters (by ids),
                     #   /dumpster_image/{id}, /dumpster/register, /dumpster/predict,
                     #   /accept_dumpster/{id}, /reject_dumpster/{id},
                     #   PUT/DELETE /dumpster/{id}, /notify_retire_dumpster
    stats.py         # /areas_dumpster_count
  services/
    mailer.py        # fastapi-mail (Gmail SMTP)
    yolo.py          # saves temp jpg under images/predicting, runs ultralytics YOLO,
                     #   returns unique class IDs; model LOADED PER CALL (known cost)
  templates/         # welcome.html, privacy.html
  images/            # dumpsters/{id}.jpg, pfps/{email}.jpg, predicting/ (temp)
  model/             # model.pt (train25 weights) + data.yaml
```

### Authentication

- `POST /login` → `{access_token, refresh_token}`. Access JWT: HS256, `sub` = user id,
  default 30 min. Refresh: opaque `secrets.token_urlsafe(64)`, stored as
  HMAC-SHA256(pepper, token) in `denouncer_refresh_token`, 90-day expiry, **max 3 rows
  per user** (oldest reused/evicted), rotated on every `/refresh`, revoked on `/logout`.
- Protected endpoints use `Depends(get_current_user)`; admin ones use `require_admin`
  (checks `denouncer.is_admin`, which can only be set manually in the DB).
- Anti-enumeration: `/forgot_password` always returns the same generic 200.

### Database (PostgreSQL 16, schema in `createDB.sql` — no migration tool, manual SQL)

| Table | Purpose |
|---|---|
| `denouncer` | users: pfp path, username, region, municipality, email (unique), paswd_hash (bcrypt via passlib), reset_token(+expiry), is_admin |
| `pending_denouncer` | pre-verification signups; PK = 64-char verify token; purged by cron when older than 1 h |
| `denouncer_refresh_token` | JWT refresh tokens (hash, created/expires/revoked) |
| `dumpster` | reports: unique `name` (ABPxxx#####), image path, lat/lon (rounded to 6 decimals), region, municipality, date, status `pending|accepted|retired`, description, denouncer FK |
| `rejected_dumpster` | trace of rejected reports (id, date, denouncer) for stats |
| `waste` | catalog seeded with IDs **0–7** matching YOLO classes |
| `dumpster_waste` | N:M link, composite PK, ON DELETE CASCADE |

Images are **not** stored in the DB — only relative paths; binary content is read from
disk and returned base64-encoded inside JSON.

### Conventions that the app depends on

- Success responses include `{"code": 200, ...}`; errors are `HTTPException` with
  appropriate status (400/401/403/404/409/500) and Catalan `detail` messages.
- `POST /dumpsters` takes `{"ids": [...]}` in the body (it used to be a GET — don't
  "restfulize" it without coordinating an app release).
- `/dumpster/register`: non-admin reports are created `pending`; admin reports are
  auto-`accepted`. Users can only register reports as themselves.
- Image uploads: base64 JPEG only (magic-byte checked), ≤ 4 MB binary / ≤ 5,000,000
  base64 chars.

### Deployment (documented step-by-step in `AbocadorsBPAPI/README.md`)

- Hetzner CX22 (2 vCPU/4 GB), Ubuntu 24.04, SSH-key-only access, Hetzner firewall
  (22 IP-restricted; 80/443 open).
- Deployed at `/srv/AbocadorsBPAPI`, Python **3.13.3** venv.
- Nginx: TLS termination (certbot/Let's Encrypt), HTTP→HTTPS redirect, serves
  `/privacy` statically, proxies the rest to Uvicorn at `127.0.0.1:8000`.
- Uvicorn as systemd unit `abocadorsbp.service`, `--workers 5`, auto-restart.
- Cron purges expired `pending_denouncer` rows every minute.
- Root-level helper scripts: `kml_to_geojson.py` (KML → GeoJSON),
  `import_dumpsters.py` (bulk import/diff → `import_all_dumpsters.sql` /
  `update_dumpsters.sql`; runs YOLO over each image), `export_dumpsters_to_excel.py`,
  `backups/` (pg dumps).

## 2. Mobile app — `AbocadorsBPApp/`

Expo SDK 54, React Native 0.81.5, React 19.1, expo-router v6 (file-based routing,
typed routes). Mostly JavaScript (`.jsx`); a few TS files from the Expo template.

```
app/
  _layout.tsx          # Theme/Geo contexts + CopilotProvider (onboarding) + Stack
  index.js             # redirect → /(tabs)/map
  (tabs)/              # map.jsx (~2900 lines), camera.jsx, profile.jsx, _layout.jsx
  login.jsx, register.jsx, edit_profile.jsx
  manage_dumpsters.jsx # admin moderation (accept/reject/edit pending reports)
  profile/[email].jsx, user_reports/[email].jsx   # dynamic routes
components/            # Text/ThemedText/ThemedView base; ImagePickerModal,
                       # ManualLocationModal, ManualWasteModal; profile/ cards & charts;
                       # ui/tutorial/ custom copilot tooltip/step
context/               # ThemeContext (light/dark, persisted '@user_theme'),
                       # AsyncGeoDataContext (lazy comarques/municipis GeoJSON),
                       # ProcessedGeoDataContext (dropdown lists + per-area counts)
constants/             # Colors.ts (tint rgb(0,150,0)), WasteTypes.js (Catalan CLASS_NAMES)
utils/
  apiWrapper.js        # apiUrl() from EXPO_PUBLIC_API_URL (prod default)
  apiClient.js         # apiFetch(): Bearer header, single-flight 401→/refresh retry,
                       #   auto-logout on refresh failure (preserves has_seen_* flags)
  tokenStorage.js      # expo-secure-store for access/refresh tokens
  api/                 # domain wrappers: auth.js, users.js, dumpsters.js (use these,
                       #   never raw fetch with hardcoded URLs)
  imageCompression.js  # expo-image-manipulator presets: PROFILE_PHOTO 512², q0.8;
                       #   DUMPSTER_PHOTO 1024×768, q0.7 (JPEG)
  validation.js        # email/password/coords/username/description validators
data/                  # comarques.json, municipis.json (slimmed GeoJSON),
                       # darkBackgroundGeoJSON.json (mask polygon)
```

### Map

- **MapLibre GL JS** loaded from CDN inside a `react-native-webview` HTML template
  (replaced Leaflet in 2026 — many identifiers still say `leafletHtml`; that's legacy
  naming, not actual Leaflet). Tiles: CARTO light/dark + OSM + Esri satellite, with
  attribution shown natively.
- RN ↔ WebView messaging: `postMessage`/`onMessage` with typed messages
  (`bboxChange`, `dumpClick`, `requestDumpsterImage`, `changeMapType`, ...).
- Data loading is viewport-driven and cached in dictionaries keyed by dumpster id:
  `bboxChange` → `GET /dumpster_ids_in_bbox` → `POST /dumpsters` for missing ids →
  `GET /dumpster_image/{id}` for visible markers only. Region/municipality filters
  draw a polygon-with-hole mask from `darkBackgroundGeoJSON` + selected geometry.

### Sessions & storage

- Tokens in **SecureStore**; everything else in AsyncStorage with `@user_*` keys
  (`@user_email`, `@user_theme`, `@user_username`, `@user_profile_picture_uri`, ...)
  plus `has_seen_{map,camera,profile}_tutorial` flags that survive logout.
- Login stores tokens + `@user_email`, then fetches profile data via `/user/{email}`.

### Builds & releases

- EAS profiles (`eas.json`): `development` (dev client, LAN API
  `http://192.168.1.103:8000`), `preview` and `production` (prod API). `autoIncrement`
  on preview/production.
- OTA updates: `expo-updates`, channel per profile, checked `ON_LOAD`,
  `runtimeVersion.policy = appVersion`.
- iOS bundle id `com.happymistery.AbocadorsBP`; Android package currently
  `com.happymistery.AbocadorsBP.preview`.

## 3. Training environment — `TFG/`

Frozen since Aug 2025; its `README.md` is accurate. Python **3.11.9** (exact), separate
venv from the API.

- `utils.py`: `prepare_data()` (downloads TACO images, converts COCO→YOLO labels),
  `split_dataset()` (70/15/15), `show_images_and_labels()`, `reclassify_labels()`.
- `labeling_tool.py` / `label_validation_tool.py`: tkinter GUIs for manual annotation
  and accept/reject validation (`preprocessement/unlabeled` → `labeled` → `data/dataset`).
- `Notebook.ipynb`: training/tuning driver (Ultralytics `model.train` / `model.tune`
  with Ray Tune; search space lr0/momentum/weight_decay; SGD; imgsz 416).
- `runs/detect/train..train26`: full training history. **train25 = production model**
  (its `weights/best.pt` is what ships as `AbocadorsBPAPI/app/model/model.pt`).
- `data/` (datasets) is populated locally, not versioned.
- Pipeline to production: train → pick best run → copy weights to
  `AbocadorsBPAPI/app/model/model.pt` (committed via Git LFS; historically also scp'd
  directly to the server) → restart the API service.
