# video-analysis

Working repo for the **VAST Builders Challenge** (video agents on a pre-running video
understanding + search stack).

Upstream starter repo: <https://github.com/yb235/vast-builders-challenge>
(kept as a gitignored clone under `vendor/vast-builders-challenge/` — read it there, don't
fork it into this repo).

## What's in here

| Path | What it is |
|------|-----------|
| `docs/stack-and-architecture.md` | The full digest: every component, every API surface, the corpus, the constraints |
| `docs/skills-capability-map.md` | All 20 skills: what each can do, its guardrails, and what nothing can do |
| `docs/approach.md` | How to attack the day: decision order, the one hard constraint, build/deploy sequence |
| `docs/upstream-map.md` | File-by-file map of the starter repo (which skill answers which question) |
| `tools/` | Our apps/clients (nothing yet) |
| `notes/` | Run notes, decisions, evidence |

## The 30-second version

- A **video ingestion + search pipeline is already running** for each team. You do not
  deploy it, and you must not rebuild its DataEngine functions.
- Storage/index = **VAST**: S3 buckets (chunks + segments) + **VastDB** (`vss-collection`
  rows: reasoning text, 256-dim text/visual vectors, YOLO detections, metadata).
- Models = **NVIDIA Cosmos3-Reason** (VLM captions/answers), **YOLO11** (detections),
  **Cosmos Embed1** (256-dim hybrid search vectors), **Canary-1B** (ASR, **not wired in**)
  — all on **CoreWeave** GPUs, called through the pipeline, not by you.
- You build against **one REST backend** (`$INGRESS_URL/api/v1/*`): login/JWT → search,
  agent Q&A, videos/playback, detections, dashboard, re-ingest.
- Your own app's reasoning runs on **Weights & Biases serverless inference** (`WANDB_*`).
- Deployment target is **the team's Kubernetes namespace**, exposed at
  `http://video-lab-team-<N>.cosmos.vastdata.com/app` (path `/app`, no Docker/registry).
- **The one hard constraint:** search can only find what the ingestion prompt asked the
  reasoner to describe. Re-ingest is how you change that.

Full detail: [`docs/stack-and-architecture.md`](docs/stack-and-architecture.md).
