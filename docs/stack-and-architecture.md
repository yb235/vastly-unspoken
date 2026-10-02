# VAST Builders Challenge — stack & architecture digest

Source: `yb235/vast-builders-challenge` @ `4987d8e` (created 2026-10-02, 34 commits),
i.e. the day-of version. Everything below is read from that repo, not assumed.

---

## 1. Layer map

```
┌ YOU BUILD HERE ─────────────────────────────────────────────────────────────┐
│  web app / CLI / agent        → VSS backend REST  /api/v1/*   (JWT)         │
│                               → W&B serverless inference (your own logic)   │
│                               → K8s Ingress /app on the team host           │
└─────────────────────────────────────────────────────────────────────────────┘
┌ PRE-RUNNING, GIVEN ─────────────────────────────────────────────────────────┐
│  VSS backend (ingress)   auth, search, agent-qa, videos, dashboard,         │
│                          metadata, re-ingest, /tools/*                      │
│  VAST DataEngine graph   Detector → Reasoner → Embedder → VastDB writer     │
│                          (+ events/prompt-suggester on a schedule)          │
│                          Segmenter already ran — NOT in the live path       │
│  VastDB                  vss-collection, vss-prompts-events (256-d vectors) │
│  VAST S3                 <team>-vss-chunks, <team>-vss-chunks-segments      │
│  GPU endpoints           Cosmos3-Reason :8001 · YOLO11 :8002 ·              │
│  (CoreWeave)             Cosmos Embed1 :8003 · Canary-1B :8004              │
└─────────────────────────────────────────────────────────────────────────────┘
```

The last four lines are "done for you". The repo's own rule: **do not rebuild or redeploy
DataEngine ingest functions**; the pipeline is the engine behind search, dashboard,
suggestions and re-ingest.

### Write path (already ran for you)

```
S3 chunks → Segmenter → S3 segments → Detector → Reasoner → Embedder → VastDB writer
```

- **Segmenter** — cut chunks into short fixed-length clips. Ran during pre-ingest; **not in
  the challenge path** (you don't upload new chunks, you don't re-segment).
- **Detector** — YOLO11 per segment → object classes, counts, bbox sidecars.
- **Reasoner** — Cosmos3-Reason writes the natural-language description of the segment.
- **Embedder** — Cosmos Embed1 → text + visual vectors (must stay **256-dim**; that's the
  VastDB `vectors` column width).
- **VastDB writer** — one row per segment: source, original_video, reasoning_content,
  vectors, vectors_visual, detections, metadata, timing. Skips duplicate `source`.
- **Events / prompt-suggester** — periodic scan of recent segments → suggested prompts and
  key events for the UI (`vss-prompts-events`).

### Read path (what you call)

Everything is one backend at `$INGRESS_URL`, all under `/api/v1`, ACL-filtered to the JWT
caller (private rows + visible public rows).

| Endpoint | Purpose |
|----------|---------|
| `POST /auth/login` → JWT, `GET /auth/me` | username/password → bearer token |
| `POST /search` | hybrid search (caption text + visual embeddings) |
| `POST /agent/ask` | grounded natural-language answer (`original_video` → scope to one video) |
| `POST /agent/search-and-answer` | same body as `/search`, returns answer **+** chunk evidence |
| `POST /tools/*` | `search`, `synthesize`; `GET` `segments`, `segment`, `detections`, `explore` |
| `GET /videos/explore` | browse fully-indexed parents (no query needed) |
| `GET /videos/stream`, `/videos/playback-url` | playback — **JWT goes in `?token=`**, not the header |
| `GET /videos/metadata`, `/videos/detections` | one segment's row / YOLO bbox sidecar |
| `POST /videos/synthesize` | LLM summary/report over one whole parent video |
| `GET /metadata/schema`, `/metadata/values` | filterable columns + autocomplete values |
| `GET /metadata/ingest-config` | **public** — scenario presets, capture types, ingest options |
| `GET /dashboard/stats` | counts, quality, objects, `s3_inventory`, `pipeline_alignment` |
| `POST /dashboard/reingest` + `GET /dashboard/reingest/<job_id>` | start / poll re-ingest |

`/search` response shape worth knowing: `results[]` = per-segment hits with
`similarity_score` + `reasoning_content`; `chunk_results[]` = hits grouped by parent upload
with `best_match_start_sec/end_sec` + `preview_source` (this is the "jump to the moment"
shape); `llm_synthesis` when hits exist; `sql_query` for debugging.

### The skill layer

`.cursor/skills/` is the intended interface — `SKILL.md` files that tell the coding agent
the endpoint, request, response, and failure modes. Groups: `ingest/`, `retrieval/`,
`deployment/`, `gpu/`. They're standard `SKILL.md`, so non-Cursor agent frameworks can load
them directly from disk. Skills read config from **absolute `/config/`** on the VM
(`/config/<team>.config`, `kubeconfig`, `backend-secret.yaml`) — never from the repo.

---

## 2. Models (all shared, on CoreWeave, no auth token in the pipeline path)

| Model | Role | Env var | Health that actually works |
|-------|------|---------|----------------------------|
| `nvidia/cosmos3-reason` | VLM: segment descriptions, backend LLM synthesis/agent answers | `$COSMOS3_REASON_URL` (:8001) | `/v1/models`, `/v1/health/ready`, `/v1/health/live` |
| `yolo11s` (Ultralytics) | detection | `$YOLO_URL` (:8002) | **`/healthz` only** (no NIM health, no `/v1/models`) |
| `nvidia/cosmos-embed1` | 256-dim text/visual embeddings | `$COSMOS_EMBED1_URL` (:8003) | `/v1/models`, `/v1/health/ready`, `/v1/health/live` |
| `nvidia/canary-1b` | ASR / speech translation (Riva HTTP) | `$CANARY_1B_URL` (:8004) | `/v1/health/ready`, `/v1/health/live` (**`/v1/models` is 404 — normal**) |

Cosmos3-Reason and Embed1 are OpenAI-ish. Embed1 uses **`request_type`** (`"query"`) instead
of OpenAI `dimensions`; video input is a `data:video/mp4;base64,...` string as `input`.
Cosmos3-Reason takes mixed content: `[{type:"text"},{type:"video_url",video_url:{url:"data:video/mp4;base64,..."}}]`.
YOLO is a custom `POST /v1/infer` with `video_base64`.

**Canary-1B is not wired into VSS.** It's the obvious "everyone else won't do this"
differentiator: transcripts, spoken-word search, subtitles. Also note Cosmos3-Reason is
callable directly — you can build a flow the pipeline doesn't have (e.g. ask a question
about a *new* clip, or a second-pass describe with a different prompt) without touching
DataEngine.

---

## 3. Corpus (pre-ingested, one index, several operational lenses)

Only `scenario` + upload metadata differ between groups. Search filters are
`camera_id`, `capture_type`, `location`, plus object class from detections.
`scenario` presets: `surveillance`, `traffic`, `live_driving`, `retail`, `warehouse`,
`egocentric`, `sports`, `nhl`, `general`.

| # | Group | Location | `camera_id` | What's indexed | Scenario |
|---|-------|----------|-------------|----------------|----------|
| 1 | Highway multi-cam | nashville | `i24_cam-1` | ~51 multi-cam clips (`scene*_p*c*`) | `traffic` |
| 2 | Live driving (PIE) | toronto | `pie_cam-3` | 6 long drive sets `set01…set06` | `live_driving` |
| 3 | Neighborhood | neighborhood | `neighborhood_cam-1` | 2 day merges (2026-09-01/02) | `surveillance` |
| 4 | SF streets | san_francisco | `sf_streets_cam-1…4` | 4 cams — **"ingesting soon"** | `surveillance` |
| 5 | Warehouse | warehouse3 | `sdg_warehouse_cam-2` | ~178 ceiling/aisle clips | `warehouse` |
| 6 | Indoor smart spaces | indoor | `smartspace_cam-1` | ~102 facility clips | `surveillance` / `crowds` |

Anchor demo line the organizers push:
> *"Show me every clip, from any camera in any site, where a person is close to a moving
> vehicle"* — should pull I-24 + Toronto driving + neighborhood + SF + warehouse in one
> result set.

Demo packs: **A** highway (I-24) · **B** live driving (PIE 01–06) · **C** warehouse (SDG) ·
**D** neighborhood · **E** SF streets (when indexed) · **F** indoor smart spaces.
Recommended starting packs: **A or C** (high signal, small) before the long day-merge sets.

---

## 4. Deployment, config, and the rules

- **Config**: single `/config/<team>.config` on the VM, already exported into the env.
  `config.example` documents every variable (`USERNAME`, `PASSWORD`, `S3_*`, `VDB_*`,
  `INGRESS_URL`, `WANDB_*`, `GPU_*`, `COSMOS*_MODEL`). Never commit real values; never
  print them (`env | cut -d= -f1 | sort` for names only).
- **App deployment** (`deployment/deploy-app-no-registry`): public image
  (`python:3.12-slim`), app code from a **ConfigMap** (~1 MiB limit), VSS creds from a
  **Secret**, Deployment + Service + **Ingress path `/app`** on the team's existing host.
  No `docker build`/`push`; a local-only server is explicitly not a valid deliverable.
  Code update = recreate ConfigMap → `rollout restart`.
- **W&B**: `WANDB_API_KEY/TEAM/PROJECT` in env; serverless LLM inference for anything your
  app decides. Credits via the W&B-by-CoreWeave team.
- **VastDB direct read** (`retrieval/vastdb-read`): `vastdb` Python SDK over an SSH tunnel
  to the data VIP, bypassing the JWT backend — for "did this row actually land?" checks.
  Exclude the vector columns when projecting.
- **Rules of the road**: stay in your team's creds/buckets/namespace · prefer skills over
  hand-rolled curl · don't redeploy infra · when search returns nothing, check login →
  dashboard/pipeline alignment → whether the metadata filter values exist · no native
  desktop/mobile apps · don't ingest internet video (licensing).

---

## 5. The constraint that shapes everything

> Every segment is described by a model following the **ingestion prompt**. Anything that
> prompt didn't ask about was never written down, so it cannot be searched later.

Consequences:

1. **The re-ingest prompt is a product decision, made before the app.** Search returning
   empty ("how many people", "what is someone carrying", "did a vehicle stop") is not a bug
   — it's a missing field in the corpus.
2. Re-ingest is expensive in wall-clock (a few minutes) and must be kicked off by **1–2
   people on the team**, not everyone, at volume.
3. Re-ingest replaces rows atomically per `(original_video, segment_number)` — row counts
   don't grow; you can't A/B two prompts on the same chunk side by side.
4. Therefore the fastest good architecture is: **prove the existing captions cover your use
   case first**, and only pay for re-ingest where they don't.

---

## 6. Open / unverified

- SF streets cameras: marked "ingesting soon" upstream — confirm in the live dashboard
  before designing around them (`GET /dashboard/stats` → `metadata` / `recent_videos`).
- The API/VastDB row does **not** persist the original `scenario` / `custom_prompt`, so you
  cannot read back which prompt produced the current captions. Evidence only.
- Two more corpus source types are listed as pending in upstream `BUILD_DAY.md`. Day-of
  check needed.
- Exact `min_similarity` behavior and `hybrid_text_weight` default are backend-side; tune
  empirically (0.3–0.5 recommended to cut noise).
