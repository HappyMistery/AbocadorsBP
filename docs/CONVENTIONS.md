# Conventions

## Cross-project

- **Language split**: user-facing strings (UI, emails, HTML pages, error `detail`
  messages) are **Catalan**. Identifiers and file names are English. Code comments are a
  Catalan/English mix — match the file you're editing. New READMEs/docs: English.
- **Three separate private GitHub repos** (`HappyMistery/AbocadorsBPAPI`,
  `HappyMistery/AbocadorsBPApp`, plus the TFG repo). Commit messages are informal,
  Catalan or English, no conventional-commits scheme. Always run git inside the
  subproject folder, never from the harness root.
- **Waste class IDs 0–7 are a contract** shared by the YOLO model (`data.yaml`), the DB
  `waste` table and the app (`constants/WasteTypes.js`). Adding a class means touching
  all three plus retraining; renumbering is forbidden.
- **No automated tests.** Changes are validated manually (dev server + dev build of the
  app). Be extra careful and conservative in shared contracts.
- **API contract stability**: endpoint paths and response shapes must stay
  backwards-compatible — installed app versions in the wild talk to the single
  production API.

## Backend (AbocadorsBPAPI)

- Python 3.13.3, venv at `.venv`, deps in `requirements.txt` (note: the file is UTF-16
  encoded; mind the encoding if you edit it).
- Layering: route handlers live in `app/routers/<domain>.py`; request validation in
  `app/schemas/`; DB access through the async `databases` instance from `app/db/base.py`
  using SQLAlchemy Core expressions (`Model.__table__.select()` etc. — no ORM sessions);
  integrations in `app/services/`; config/auth plumbing in `app/core/`.
- All config via `.env` + `app/core/config.py` constants. Never hardcode secrets or URLs;
  build links with `apiUrl()` from `app/core/api.py`.
- Validation/sanitization belongs in Pydantic `field_validator`s: `html.escape()` text,
  enforce length limits matching the DB schema (region ≤ 17, municipality ≤ 43,
  description ≤ 500, username ≤ 100), base64-JPEG checks for images (magic bytes
  `\xff\xd8\xff`, ≤ 4 MB).
- Responses: success returns a dict with `"code": 200` plus payload; failures raise
  `HTTPException` with Catalan `detail`. Endpoints that browsers hit (verify/reset
  flows) return `HTMLResponse`, content negotiated via the `Accept` header where both
  apply.
- Auth: protect user actions with `Depends(get_current_user)` and admin actions with
  `Depends(require_admin)`. Public read endpoints (map data, public profiles) stay
  unauthenticated by design.
- Images: write to `app/images/{dumpsters,pfps}` with deterministic names
  (`{dumpster_id}.jpg`, `{email}.jpg`); store only the relative path in the DB.
- Schema changes: edit `createDB.sql` **and** apply matching manual SQL in production
  (no Alembic). Keep `app/db/models.py` in sync.
- Keep Swagger disabled in production (`docs_url=None, redoc_url=None, openapi_url=None`).

## Mobile app (AbocadorsBPApp)

- Expo-managed workflow; screens are `.jsx` files under `app/` (expo-router file-based
  routing; tabs in `app/(tabs)/`, dynamic routes like `profile/[email].jsx`).
- **All network calls go through `utils/api/{auth,users,dumpsters}.js`**, which wrap
  `apiFetch()` (`utils/apiClient.js`). Never call `fetch` with a hardcoded URL —
  `apiFetch` attaches the Bearer token, retries once after a 401 by refreshing the
  session, and signs out cleanly if refresh fails. API wrappers return
  `{ ok, status?, message?, data }` objects rather than throwing.
- Tokens only in `expo-secure-store` (`utils/tokenStorage.js`). Other persisted state in
  AsyncStorage: `@user_*` keys for profile/theme cache, `is_admin`, and
  `has_seen_*_tutorial` flags (these must survive logout — see
  `clearLocalSessionData()`).
- Theming: use `ThemeContext` + `Colors` from `constants/Colors.ts`
  (tint `rgb(0,150,0)`); base components `Text`, `ThemedText`, `ThemedView` instead of
  raw RN `Text`/`View` (font scaling is intentionally disabled).
- Responsive sizing via `react-native-size-matters` (`scale`/`verticalScale`/
  `moderateScale`) — don't hardcode pixel sizes in new UI.
- Compress images before upload with `compressImage` + `COMPRESSION_PRESETS`
  (`PROFILE_PHOTO`, `DUMPSTER_PHOTO`); the API rejects non-JPEG or > 4 MB.
- Map work happens inside the WebView HTML template strings in `app/(tabs)/map.jsx` and
  `components/ManualLocationModal.js`. The map engine is **MapLibre GL JS** (CDN);
  identifiers named `leaflet*` are legacy. RN↔WebView messages are JSON with a `type`
  field — extend the existing message protocol rather than adding new channels.
- Large screens are intentionally monolithic (`map.jsx` ~2.9k lines) with banner-comment
  sections (`// ====== IMPORTS ======`). Follow the section structure; extract to
  `components/` only when something is reused.
- Validation helpers in `utils/validation.js`; show user errors with `Alert.alert`
  (Catalan copy).
- Lint with `npm run lint` (eslint-config-expo) and dead-code check with `npm run knip`.
- Builds via EAS (`npm run build:dev|preview|production`); dev client uses the LAN API
  URL from `eas.json` — update that IP if your dev machine changes. OTA via
  `expo-updates`; bump `version` in `app.json` for native releases
  (`runtimeVersion.policy = appVersion` means OTA only reaches same-version builds).

## Training env (TFG)

- Python 3.11.9 exact, own venv, deps in `requirements.txt` (note `sympy==1.12` pin).
- Dataset flow: `prepare_data()` → manual labeling (`labeling_tool.py`) → validation
  (`label_validation_tool.py`) → `split_dataset()` → train via `Notebook.ipynb`.
- YOLO label format: `class x_center y_center width height` (normalized); classes
  defined in `data.yaml`.
- Training runs accumulate under `runs/detect/trainN`; production weights come from the
  chosen run's `weights/best.pt` and are copied into the API repo as
  `app/model/model.pt`.
