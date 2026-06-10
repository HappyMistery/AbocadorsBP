# AbocadorsBP — Harness

AbocadorsBP is a citizen platform for reporting and monitoring illegal dumping sites
("abocadors") in the Baix Penedès region (Catalonia). It started as Jaume Tello Viñas's
TFG (bachelor's thesis, URV, 2025) and is now a **live product in production** at
`https://abocadorsbp.com`, with the mobile app published on the App Store and in testing
on Google Play.

This root folder is **not** a code repository. It is the harness/meta repo: it holds
project documentation and wraps three sibling projects, each of which is its **own git
repository** (gitignored here):

| Folder | What it is | Stack |
|---|---|---|
| `AbocadorsBPAPI/` | Backend REST API + AI inference | Python 3.13 / FastAPI / PostgreSQL / YOLOv11 |
| `AbocadorsBPApp/` | Mobile app (iOS + Android) | React Native / Expo SDK 54 / expo-router |
| `TFG/` | Training environment for the waste-detection model | Python 3.11 / Ultralytics YOLO / Ray Tune |
| `Documentació TFG/` | Thesis documents (PDF, slides, demo video) | — |

## Documentation map

Read these before making non-trivial changes:

- [docs/CONTEXT.md](docs/CONTEXT.md) — what the project is, its history and current status.
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — verified system architecture (app ↔ API ↔ DB ↔ model, deployment).
- [docs/CONVENTIONS.md](docs/CONVENTIONS.md) — code, naming, language and workflow conventions.
- [docs/TFG-VS-CODE.md](docs/TFG-VS-CODE.md) — where the thesis PDF is **outdated** vs the actual code. Check this before citing the PDF as a source.

## Critical warnings

1. **Stale documentation exists.** The TFG PDF (`Documentació TFG/TFG_TelloViñas_Jaume.pdf`)
   describes the system as of ~mid-2025 and several core decisions have changed since
   (JWT auth, MapLibre instead of Leaflet, Nginx in front of Uvicorn, layered backend...).
   `AbocadorsBPApp/README.md` is also partially stale (mentions Expo 53/Leaflet/`utils/api.js`).
   **The code is the source of truth**; `AbocadorsBPAPI/README.md` is current and reliable.
2. **Production is live.** The API serves real users and a real PostgreSQL database on a
   Hetzner server. Be conservative with anything touching `createDB.sql`, endpoint paths
   or response shapes — the deployed app (including older installed versions) depends on them.
3. **Secrets.** `AbocadorsBPAPI/.env` exists locally and holds real credentials (DB, Gmail
   SMTP, JWT secret, refresh-token pepper). Never read it into output, commit it, or copy
   values from it into docs.
4. **Three separate repos.** Run git commands inside the subproject you are changing, not
   from this root. This root repo only tracks the harness docs and `Documentació TFG/`.
5. **Languages.** User-facing strings, emails and most code comments are in **Catalan**.
   Identifiers are in English. Keep that split; don't translate existing UI text.

## Quick facts

- Waste classes (IDs are shared by the YOLO model, the DB `waste` table and the app's
  `CLASS_NAMES`): 0 Metal, 1 Plastic, 2 Glass, 3 Paper & Cardboard, 4 Rubble,
  5 Organic Waste, 6 Other/Miscellaneous, 7 Furniture. **Never renumber.**
- Dumpster public codes: `ABP` + 3-letter municipality code + 5-digit counter
  (e.g. `ABPTGN00001`); fallback prefix `ABPUNK`. Mapping lives in
  `AbocadorsBPAPI/comarques_municipis.json`.
- API base URL: prod `https://abocadorsbp.com:443`; configurable via `EXPO_PUBLIC_API_URL`
  (app, `eas.json` profiles) and `PUBLIC_API_URL`/`API_BASE_URL`/`EXPO_PUBLIC_API_URL` (API).
- No automated tests anywhere; verification is manual. The app has `npm run lint` (eslint)
  and `npm run knip`.
