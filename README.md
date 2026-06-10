# AbocadorsBP

> ⚠️ **This repository contains no application code.** It exists only to hold the
> project **harness** (AI-assistant / contributor documentation) and general **project
> information** for AbocadorsBP. The actual code lives in separate, private git
> repositories that sit in sibling folders and are deliberately gitignored here.

AbocadorsBP is a citizen platform for reporting and monitoring illegal dumping sites in
the Baix Penedès region (Catalonia): a mobile app to photograph and geolocate dumps, an
AI model that classifies the waste in the photo, and a backend that centralizes reports
on an interactive map with moderation and statistics. It started as a bachelor's thesis
(TFG, URV 2025) and is now live in production at <https://abocadorsbp.com>.

## What this project encompasses

| Component | Folder | Description |
|---|---|---|
| **AbocadorsBPAPI** | `AbocadorsBPAPI/` | The **backend** of the project: a FastAPI REST API with PostgreSQL persistence, JWT authentication, email automation and integrated YOLOv11 waste detection. Deployed on a Hetzner server behind Nginx. |
| **AbocadorsBPApp** | `AbocadorsBPApp/` | The **frontend** of the project: a cross-platform mobile app (React Native + Expo) with an interactive MapLibre map, camera-based report flow, user profiles/statistics and admin moderation screens. |
| **TFG** | `TFG/` | The **environment used to train the waste detection model**: dataset preparation (TACO + Roboflow), custom labeling/validation tools, and Ultralytics YOLOv11 training with Ray Tune hyperparameter optimization. |

Each of those folders is its own git repository — clone/commit/push inside them, not here.

`Documentació TFG/` holds the original thesis documents (PDF, slides, requirements,
demo video). Note that the thesis describes the project as of 2025 and is **not** the
source of truth for the current implementation.

## Harness / documentation

- [CLAUDE.md](CLAUDE.md) — entry point for AI-assisted work: structure, warnings, quick facts.
- [docs/CONTEXT.md](docs/CONTEXT.md) — project background, history and current status.
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — system architecture, verified against the code.
- [docs/CONVENTIONS.md](docs/CONVENTIONS.md) — code, naming, language and workflow conventions.
- [docs/TFG-VS-CODE.md](docs/TFG-VS-CODE.md) — where the thesis PDF diverges from the actual code.
