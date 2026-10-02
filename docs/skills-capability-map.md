# Skills capability map

All 20 skills in `vendor/vast-builders-challenge/.cursor/skills/`, read at `4987d8e`. Grouped by
what they let you *do*, with the guardrails each one enforces.

---

## A. Authenticate / establish a session

**`retrieval/login`** — `POST /auth/login` `{username,password}` → `{access_token, token_type:"bearer", username}`.
- Verifies with `GET /auth/me` → `{username,email,auth_type}`.
- Backend authenticates against VMS; host + tenant come from backend config, not the request.
- Token lifetime is backend-configured; on 401 mid-session, re-login.
- **Every subsequent result is ACL-filtered to this user** (their private rows + visible public rows).
- Two auth systems exist and must not be conflated: this backend JWT vs. raw VastDB SDK keys.

## B. Find moments (retrieval)

**`retrieval/search`** — `POST /search`, the workhorse.
- Hybrid: caption-text embeddings **+** visual embeddings over `vss-collection`.
- Inputs: `query` (natural language), `top_k` 1–100 (default 15), `llm_top_n` (default 3 chunks sent to the synthesizer), `min_similarity` (default 0.1; 0.3–0.8 advised), `tags[]`, `time_filter` (`5m/15m/1h/24h/7d/custom` + ISO date bounds), `metadata_filters{}` (column→value), `include_public`/`public_only`, `hybrid_text_weight` (caption-vs-visual blend), `system_prompt` (override the synthesis prompt).
- Outputs four things: `results[]` (per-segment hits with `source`, `similarity_score`, `reasoning_content`, timings), `chunk_results[]` (hits grouped by parent upload with `best_match_start_sec/end_sec`, `preview_source`, `matched_segment_count`), `llm_synthesis` (`{response, segments_used, model, tokens_used}`), and `sql_query` (the VastDB query actually executed — a debug gift).
- It **cannot** discover its own filters: there is no `/tags` or `/locations` route, and the skill says to parse phrasing into filters client-side and resolve values via `list-metadata`.
- Same engine also exposed as `POST /tools/search` and, with an answer, `POST /agent/search-and-answer`.

**`retrieval/agent-qa`** — answer, not hit list.
- `POST /agent/ask` `{question, original_video?, top_k}` → `{answer, tool_used: "search_hybrid"|"video_segments", evidence}`. Set `original_video` to scope the answer to one parent video.
- `POST /agent/search-and-answer` = search body + grounded answer + rich chunk evidence.
- `/tools/*` exposes the primitives for your own orchestration: `POST tools/search`, `POST tools/synthesize`, `GET tools/segments?original_video=`, `GET tools/segment?source=`, `GET tools/detections?source=`, `GET tools/explore`.
- Explicit negative: **there is no `/videos/ask`**.

**`retrieval/videos`** — browse, play, inspect, summarise.
- `GET /videos/explore?scope=all|mine|public&limit&offset&date=YYYY-MM-DD&location=` — lists **fully-indexed** parents only, no query needed. (Useful side property: anything Explore shows is safe to re-ingest whole.)
- `GET /videos/stream?source=s3://…&token=JWT` — range-capable, seekable proxy stream. `GET /videos/playback-url?…&expires_in=` — presigned S3 URL (needs bucket CORS for browser playback). **Both take the JWT as a query param**, because `<video>` can't set headers.
- `GET /videos/metadata?source=` — one segment's row (reasoning, tags, timing). `GET /videos/detections?source=` — YOLO bbox sidecar JSON; 404 means no sidecar, not an error.
- `POST /videos/synthesize` `{original_video, question?, max_segments?, system_prompt?}` — LLM summary over **all** segments of one parent, chronological Overview + Timeline by default, **no VastDB write**, 404 if no accessible segments. This is the only "report" capability.

**`retrieval/suggest-prompts`** — `GET /suggestions` → ready-to-run prompt chips + notable "key events", produced by the scheduled `prompt-suggester` enrichment job writing to `vss-prompts-events`. ACL-filtered. Empty means the job hasn't run or the lookback found nothing — check the dashboard, don't assume a bug.

**`retrieval/list-metadata`** — the filter resolver, three routes:
- `GET /metadata/schema` → filterable columns with type, label, `ui_type` (`select` + `options` if ≤100 distinct values, else `text`).
- `GET /metadata/values?field=&prefix=&limit=` → value autocomplete. Non-filterable field → 400.
- `GET /metadata/ingest-config` → **public, no auth**; canonical scenario presets, capture types, labels, `custom_prompt_max_length` (800).
- Hard limits it documents: internal columns (`pk`, `vectors`, `vectors_visual`, `source`, `reasoning_content`, `perception_json`, timing, `tags`, `is_public`) are **excluded** and can't be filtered. Schema/values are for *search*; ingest-config is for *ingest* — don't cross them.

**`retrieval/dashboard`** — `GET /dashboard/stats?scope=all|mine|public`, one bundled aggregate (all sections returned regardless of scope):
- `overview` (total_rows, segment_rows, unique_videos, fully_indexed_videos, indexed_clips, re_ingest_rows, stream_sessions, duplicate_segment_rows, public/private), `quality` (reasoning_ok_pct, perception_ok_pct, with_object_classes_pct), `objects[]` (label, segment_count, instance_count), `metadata{}` (value distributions per upload field), `uploads_by_day[]`, `recent_videos[]` (per parent incl. `unique_segments` vs `expected_segments`), `s3_inventory` (chunks/segments/segmenter output counts), `pipeline_alignment` (S3 vs indexed, `pending_index`, `healthy`).
- Doubles as the ingest-health diagnostic: it tells you the difference between "not indexed yet" and "silently dropped".

## C. Change what's searchable (ingest)

**`ingest/reingest-videos`** — re-run an indexed video/stream through Detector → Reasoner → Embedder → writer with a different prompt/metadata. The challenge's main lever.
- `POST /dashboard/reingest` `{stream_id, chunk_count}` or `{original_video, chunk_count:1}`, plus optional `camera_id`, `capture_type`, `location`, `scenario`, `custom_prompt`.
- **Enforced interaction contract**: it must not start until the user has chosen (1) target, (2) preserve-or-override prompt/metadata, (3) number of latest complete chunks, (4) confirmed. It's explicitly told to never guess missing choices.
- It fetches *all* Explore pages (not just the dashboard's 15 recent) and groups chunks by `stream_id`, excluding incomplete chunks (a chunk is complete only when its timeline has every segment 1..total_segments).
- Honesty constraints: the row doesn't persist the old `scenario`/`custom_prompt`, so it must say so rather than display or invent them; "keep original prompt" means the canonical original S3 segments, which may differ from the immediately preceding override.
- Prompt semantics: a custom prompt **overrides** a scenario; sending a new scenario without a custom prompt **intentionally removes** the old custom prompt; blank metadata fields mean *preserve*, not erase.
- `chunk_count` is 1–min(available complete chunks, 100); re-ingest always takes whole chunks, never arbitrary clips; streams mean "latest N chunks".
- Monitoring: poll `GET /dashboard/reingest/<job_id>` every 4s, report `completed/total chunks` + `indexed_segments/total_segments`, finish only on `completed`. A 404 after a backend restart only means the in-memory progress record was lost — not that processing stopped.
- Validation step: refresh dashboard/explore and confirm the target still has the expected clip count; row counts should **not** grow (atomic replacement per `(original_video, segment_number)`).

**`ingest/reingest-chunk`** — same write, one Explore card.
- Finds the card from a filename (completing `s3://$S3_CHUNKS_BUCKET/$USERNAME/<filename>`), a truncated title, or a date/location/camera/capture_type/description-of-scene. For semantic discovery it searches with `llm_top_n: 0` and uses `chunk_results`.
- Must never silently pick the first match — ambiguous → show candidates and ask.
- Never sends `stream_id` (that would mean "latest N of the session" and could hit a different card).

**`ingest/upload-video`** — add genuinely new content.
- `POST /videos/upload`, `multipart/form-data`: `file`, `is_public`, `tags`, `allowed_users`, `scenario`, `custom_prompt`, `camera_id`, `capture_type`, `location`.
- Accepted: `.mp4 .mov .webm .avi .mkv`; size cap read live from `GET /api/v1/config` → `app.max_upload_size_mb` (typically 25–100 MB). One file per request; loop and report per file.
- Full pipeline runs (segmenter → detector → reasoner → embedder → writer); async, tens of seconds to minutes. Returns `object_key`.
- Same interaction contract (file, visibility, prompt mode, metadata, confirmation). Must not claim searchability until explore/dashboard shows the clips.
- Rules of the road forbid internet-sourced video (licensing).

## D. Go around the API

**`retrieval/vastdb-read`** — raw store access with the `vastdb` Python SDK over an SSH tunnel (`ssh -N -f -L 18080:<vip>:80 …`), **bypassing the JWT backend and its ACLs**.
- VSS defaults: bucket `vss-db`, schema `vss-schema`, table `vss-collection` (rows: `source`, `original_video`, `reasoning_content`, `vectors` 256-d text, `vectors_visual`, metadata, timing); plus `vss-prompts-events`.
- Ships `list_catalog.py`: parses `/config/*.config`, resolves endpoint (`VDB_ENDPOINT` → `S3_ENDPOINT`), probes the tunnel with a helpful error, prints database → schema → tables from `tx.catalog()`, skipping internal tables (`tabular_schema_table`), with an optional live `bucket.schemas()` drill-down. Supports `--bucket` and `--live-only`.
- Row reads: project columns but **exclude `vectors`/`vectors_visual`** — fixed-size list select can fail without the pipeline's `common/vastdb_patch.py`.
- Diagnostics it enables: row count + recency, "did this exact clip get written", duplicate `source` detection, and vectors length **must be 256**.

## E. Call the models directly (all four, without the pipeline)

**`gpu/README.md` + `gpu/model-health` + `gpu/model-smoke-test`** — the endpoints are callable directly; the pipeline is not the only consumer.
- **Cosmos3-Reason** :8001 — OpenAI-compatible `POST /v1/chat/completions`, and content can mix `{type:"text"}` with `{type:"video_url", video_url:{url:"data:video/mp4;base64,…"}}`. So you can ask a question about an arbitrary clip the index has never described.
- **YOLO11** :8002 — `POST /v1/infer` `{video_base64, filename, include_frames}` → `perception_ok`, `object_classes`, `object_counts`, `frames`. Custom FastAPI, **`/healthz` only**.
- **Cosmos Embed1** :8003 — `POST /v1/embeddings` with Cosmos-style `request_type:"query"` (not OpenAI `dimensions`); text or `data:video/mp4;base64` input; must return **256** dims.
- **Canary-1B** :8004 — `POST /v1/audio/transcriptions` (multipart `file`) → ASR / speech translation. Riva HTTP; **`/v1/models` 404 is expected**; `ready`/`live` only. Not wired into the pipeline.
- Health is per-NIM and enforced as such (a documented matrix, with the explicit instruction not to probe the same paths on every model, and that 404 on an unsupported path is not a failure).
- Smoke tests assert real inference: non-empty `choices[0].message.content`, `dim == 256`, YOLO `ok`/`model_loaded`, Canary ready/live.
- Auth: `GPU_BEARER_TOKEN` from `/config/<team>.config` (note: `config.example` claims no token is needed — a doc conflict worth testing).

## F. Ship and get help

**`deployment/deploy-app-no-registry`** — put your app on the cluster with no Docker/registry: public image (`python:3.12-slim`, or `node:22-slim`), code from a **ConfigMap** (~1 MiB soft limit), VSS creds from a **Secret**, Deployment + Service + **Ingress path `/app`** on the team's existing host with an nginx rewrite, readiness probe on `/health`. Includes the update loop (recreate ConfigMap → `rollout restart`) and a pitfall table. Non-negotiables: on-cluster, `/app` path, one host, own namespace only, never a local server as the deliverable.

**`deployment/build-yamls` / `deploy` / `health`** — for the official retrieval stack (backend :8000, frontend :80, batch-sync): fill `backend-secret.yaml` (jwt_secret mandatory or the backend won't boot; embedding dims 256; host/ports 166.19.38.112:8001/8003), set image tags, `QUICK_DEPLOY.sh <ns> <cluster>`, then pods/`/health`/`GET /api/v1/config`. Not needed for a mini-app, but it is the reference for the coupling rules (backend underscore keys vs ingest no-underscore keys; **VastDB endpoint intentionally differs** — Query Engine VIP for reads vs data VIP for writes).

**`ask-cosmos`** — runs the health check, classifies the three common blockers (stack unhealthy / filter value doesn't exist / prompt never described it), and produces a ≤120-word note in a fixed 6-line layout, **redacted** (URLs, IPs, tokens, bucket/collection names, emails all replaced; the team name is the one exception). It never posts anywhere.

**`submission`** — reads your code to *derive* the project description (≤40 words: what it does + who it's for) and the stack (skills called → implied pipeline parts, whether a W&B model was used, what the app is built with), asks for links/feedback one section at a time, never collects personal details, never invents a value, and writes `SUBMISSION.md` without committing it.

---

## What the skill set as a whole can and cannot do

**Can:** authenticate; semantic + filtered search over video; grounded Q&A with evidence; browse/play/inspect clips including YOLO boxes; summarise a whole video; discover filter vocabulary; read index health; re-describe existing footage under a new prompt; add new local footage; read the database directly; call all four models directly; deploy a small app on the cluster; write a support note; draft a submission.

**Cannot (no skill touches these):** the DataEngine graph (rebuild, add functions, redeploy the pipeline, re-run the Segmenter); delete or edit rows directly; any alerting/notification/webhook capability; any streaming or live-camera ingest; `/reports`, `/alerts`, `/analytics`, `/tags`, `/locations`, `/videos/ask` routes; cross-team access; readback of the prompt that produced existing captions; scheduling or background jobs of your own; native desktop/mobile delivery.

**Design consequence:** the platform gives you *read + re-describe + raw access + deploy*. Anything that "acts" on what it finds — thresholds, alerts, notifications, escalations, cross-camera correlation — is code you write (typically in the deployed app, using W&B inference for judgement). That gap is where the differentiating demos live.
