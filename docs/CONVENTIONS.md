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

- Python 3.13.3, venv at `.venv`, deps **pinned** in `requirements.txt` (UTF-8). JWT uses
  `PyJWT` — don't reintroduce `python-jose` (migrated away 2026-06 over advisories).
- Layering: route handlers live in `app/routers/<domain>.py`; request validation in
  `app/schemas/`; DB access through the async `databases` instance from `app/db/base.py`
  using SQLAlchemy Core expressions (no ORM sessions);
  integrations in `app/services/`; config/auth plumbing in `app/core/`.
- **Minimal SELECT**: queries select only the columns the endpoint actually uses
  (`select(Model.col1, Model.col2)`) — never `Model.__table__.select()` full rows.
  `paswd_hash` and the per-flow token columns only ever appear in the specific auth flow
  that needs them. Shared helpers: `get_user_or_404(email, *columns)` in
  `app/db/queries.py`, `image_to_base64()` in `app/services/images.py`.
- All config via `.env` + `app/core/config.py` constants. Never hardcode secrets or URLs;
  build links with `apiUrl()` from `app/core/api.py`.
- Validation/sanitization belongs in Pydantic `field_validator`s: `html.escape()` text,
  enforce length limits matching the DB schema (region ≤ 17, municipality ≤ 43,
  description ≤ 500, username ≤ 100), base64-JPEG checks for images (magic bytes
  `\xff\xd8\xff`, ≤ 4 MB).
- Responses — **the HTTP status code is authoritative**. Success is HTTP 200 with a dict
  body `{"code": 200, ...payload}` (the `code` field is legacy — installed apps check it —
  and is only ever 200). Every failure raises `HTTPException` with the proper 4xx/5xx
  status and a Catalan `detail`; never return an error-shaped body (`code` ≠ 200) under
  HTTP 200. Never put exception text (`str(e)`) in a response body — log details with
  `logger.exception`, return a generic Catalan message. Endpoints that browsers hit
  (verify/reset flows) return `HTMLResponse`, content negotiated via the `Accept` header
  where both apply.
- Auth is **default-on**: the global dependency `require_auth_by_default`
  (`app/core/auth_deps.py`) demands a valid access token on every route whose
  `(method, path)` is not in the explicit `PUBLIC_PATHS` allow-list. The list is resolved
  against the registered routes at startup and the app refuses to boot on a stale entry,
  so a new endpoint is private until deliberately added to `PUBLIC_PATHS`. Handlers that
  use the user still declare `Depends(get_current_user)` / `Depends(require_admin)`
  explicitly (the gate caches the resolved user on `request.state`, so nothing is looked
  up twice). Public read endpoints (map data, public profiles) stay unauthenticated by
  design — via the allow-list.
- **No emails in URLs / public responses.** The unauthenticated profile surface is
  id-keyed and PII-free (`/public_profile/{id}/*`; `/user_info/{id}` returns username
  only) — it never exposes email or `is_admin`. Email-keyed user endpoints are
  authenticated **self-only**: reject with a generic 403 *before any DB lookup* when the
  email isn't the caller's own (`_require_self` in `users.py`), so they can't be used as
  an account-existence oracle. Don't add new email-keyed endpoints.
- Images: write to `app/images/{dumpsters,pfps}` with deterministic names
  (`{dumpster_id}.jpg`, `{email}.jpg`); store only the relative path in the DB.
- Schema changes: edit `createDB.sql` **and** apply matching manual SQL in production
  (no Alembic). Keep `app/db/models.py` in sync.
- Keep Swagger disabled in production (`docs_url=None, redoc_url=None, openapi_url=None`).

## Mobile app (AbocadorsBPApp)

- Expo-managed workflow; screens are `.jsx` files under `app/` (expo-router file-based
  routing; tabs in `app/(tabs)/`, dynamic routes like `profile/[id].jsx`). Profile
  navigation is keyed on the denouncer **id**, never the email.
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
  (Catalan copy). `isReservedUsername()` mirrors the server's reserved-username rules
  (`admin`/`root`/`system`/`self` for everyone; `AbocadorsBP` for non-admins) — keep it
  in sync with `schemas/users.py` + `/edit_user`; the server stays authoritative.
- Lint with `npm run lint` (eslint-config-expo) and dead-code check with `npm run knip`.
- Builds via EAS (`npm run build:dev|preview|production`); dev client uses the LAN API
  URL from `eas.json` — update that IP if your dev machine changes. OTA via
  `expo-updates`; bump `version` in `app.json` for native releases
  (`runtimeVersion.policy = appVersion` means OTA only reaches same-version builds).

## Web (AbocadorsBPWeb)

- Next.js App Router, TypeScript strict, Tailwind v4 (`@theme` palette in
  `src/app/globals.css` mirrors the app's `Colors.ts`; tint `rgb(0,150,0)`). Shared
  component classes (`btn-primary`, `card`, `input`, …) live in `@layer components`.
- **All network calls go through `src/lib/api/{auth,users,dumpsters,blog,admin}.ts`**,
  which wrap `apiFetch()` (`src/lib/api/client.ts`) — same contract as the app's
  wrappers: return `{ ok, status, message?, data? }`, never throw. Browser calls hit
  `/api/*` (proxy route handler `src/app/api/[...path]/route.ts`); server components
  use `src/lib/serverApi.ts`. Both resolve the API URL through `apiTarget()`
  (`API_PROXY_TARGET` in `.env`, read at runtime) — never hardcode the API origin.
- Session: tokens + email in `localStorage` via `src/lib/tokens.ts` only; user state
  through `useAuth()` (`src/lib/AuthContext.tsx`). Admin UI is gated client-side in
  `src/app/admin/layout.tsx`, but the API's `require_admin` is the real gate.
- Compress images in the browser before upload (`src/lib/image.ts`, same presets as
  the app); the API only accepts JPEG ≤ 4 MB.
- Blog content is Markdown rendered exclusively with `react-markdown` (no
  `dangerouslySetInnerHTML`) — this is what makes storing unescaped Markdown safe;
  don't change one without the other.
- Editable site facts (donation links, store links, cost table, contact email) live
  in `src/lib/config.ts`, not in page copy.
- Lint with `npm run lint`; `npm run build` type-checks. No automated tests.

## Training env (TFG)

- Python 3.11.9 exact, own venv, deps in `requirements.txt` (note `sympy==1.12` pin).
- Dataset flow: `prepare_data()` → manual labeling (`labeling_tool.py`) → validation
  (`label_validation_tool.py`) → `split_dataset()` → train via `Notebook.ipynb`.
- YOLO label format: `class x_center y_center width height` (normalized); classes
  defined in `data.yaml`.
- Training runs accumulate under `runs/detect/trainN`; production weights come from the
  chosen run's `weights/best.pt` and are copied into the API repo as
  `app/model/model.pt`.
