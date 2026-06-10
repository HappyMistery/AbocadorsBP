# Project context

## What AbocadorsBP is

AbocadorsBP ("Abocadors Baix Penedès") is a citizen-driven platform for detecting,
reporting and monitoring **illegal dumping sites** in the Baix Penedès region of
Catalonia. A user photographs a dump; the app geolocates it automatically, a YOLOv11
model detects which waste types appear in the photo, and the report is stored in a
central database. Reports are explored on an interactive map, moderated by
administrators, and surfaced through per-user and per-area statistics.

## Origin and people

- Author/owner: **Jaume Tello Viñas** (GitHub `HappyMistery`). The project was his
  Treball de Fi de Grau (Computer Engineering, Universitat Rovira i Virgili, Tarragona,
  2025), supervised by Usama Benabdelkrim Zakan.
- The idea comes from his father, who had been manually documenting illegal dumps in the
  Baix Penedès for years; that manual collection (~1,080 sites in KML) became the
  initial database import.
- All code lives in private GitHub repos under `HappyMistery` (kept private for security
  and possible future commercial exploitation).

## Current status (as of June 2026)

- **In production**: API live at `https://abocadorsbp.com` on a Hetzner VPS, serving the
  mobile app 24/7.
- **iOS**: published on the App Store ("abocadorsbp"). **Android**: distributed through
  Google Play (closed testing track at thesis time; the Android application id currently
  carries a `.preview` suffix — `com.happymistery.AbocadorsBP.preview`).
- App version `1.3.0` (see `AbocadorsBPApp/app.json`), with OTA updates enabled via
  `expo-updates` (checked on app load).
- Active development continues post-thesis: recent work includes the JWT auth overhaul,
  the switch from Leaflet to MapLibre GL JS, email-change and account-deletion flows,
  onboarding tutorials (react-native-copilot), and a Nginx-fronted deployment.
- The thesis mentions planned contact with local administrations (town councils,
  comarcal council) to turn the data into institutional action.

## The three components

1. **AbocadorsBPAPI** — FastAPI backend: auth, user accounts, dumpster reports and
   moderation, statistics, email automation, and YOLO inference. PostgreSQL persistence.
2. **AbocadorsBPApp** — React Native + Expo app: map (MapLibre in a WebView), camera
   report flow with AI-assisted waste classification, profile with stats/charts, admin
   moderation screens.
3. **TFG** — the model-training environment: dataset preparation (TACO + Roboflow
   datasets, ~15k images), custom tkinter labeling/validation tools, Ultralytics
   training, Ray Tune hyperparameter search. Produced the production model
   (`runs/detect/train25` → copied to `AbocadorsBPAPI/app/model/model.pt`).
   Essentially frozen since Aug 2025; it still matches the thesis description.

## Model at a glance

- YOLOv11x fine-tuned by transfer learning on a ~14,974-image dataset (TACO + several
  Roboflow datasets), 70/15/15 split, imgsz 416, SGD with Ray Tune-optimized
  lr0/momentum/weight_decay.
- Production metrics (train25): ~55% mAP@50-95, ~68% mAP@50, P ~74.5%, R ~62%.
  Strong classes: Furniture (~0.99 AP@0.5), Glass, Organic; weak: Rubble (~0.51),
  Other/Misc (~0.07, heterogeneous + underrepresented).
- 8 classes whose IDs are shared verbatim across model, DB and app (see CLAUDE.md).
- Future intentions from the thesis (not yet realized): retrain with user-contributed
  data, class rebalancing, YOLOv12, a web portal (abocadorsbp.com) embedding the
  labeling/validation tools for crowdsourced annotation.

## Domain vocabulary

- *Abocador* = dumping site / dumpster (the DB entity is `dumpster`).
- *Denouncer* = a user who reports dumps (DB table `denouncer`).
- *Comarca* = county-level region (e.g. "Baix Penedès"); *municipi* = municipality.
  Max lengths 17 and 43 chars are derived from the longest real Catalan names.
- Report lifecycle: `pending` → `accepted` | rejected (moved to `rejected_dumpster`) →
  `retired` (dump cleaned up; users can request retirement, admin confirms).
