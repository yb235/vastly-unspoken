# Upstream starter repo — where to look for what

Clone (gitignored, local only):

```bash
git clone https://github.com/yb235/vast-builders-challenge.git vendor/vast-builders-challenge
```

Pinned read: `4987d8e` · 34 commits · repo created 2026-10-02.

## Top-level docs

| File | Read it for |
|---|---|
| `README.md` | 1-screen orientation + links to the three guides (PDF fallbacks in `docs/`) |
| `BEFORE_YOU_BUILD.md` | Pre-event checklist: Cosmos Community, Cursor signup (personal email), W&B account, team (≤4), the stack blurb, use-case framing |
| `BUILD_DAY.md` | The day itself: kickoff, launch VM, team select, skills tour, test drive loop, build, demo/submission, health check |
| `ARCHITECTURE_REFERENCE.md` | Models table, DataEngine graph, what you were given, **full corpus** + packs + example queries, suggested playbook prompts, rules |
| `config.example` | Every env var and what it means — including the `/config` aliases and the discovered-model-id fallbacks |

## `.cursor/`

| Path | What it holds |
|---|---|
| `rules/build-day.mdc` | `alwaysApply` ground rules: use a skill before raw REST, config is pre-set, never print secrets, web app only, `ask-cosmos` when stuck, `submission` when done |
| `skills/retrieval/login` | JWT from `/auth/login`; playback uses `?token=` |
| `skills/retrieval/search` | `POST /search` field table + how to build filters client-side |
| `skills/retrieval/agent-qa` | `/agent/ask` vs `/agent/search-and-answer` vs `/tools/*` — when to use which |
| `skills/retrieval/videos` | explore / stream / playback-url / metadata / detections / synthesize |
| `skills/retrieval/list-metadata` | `schema`, `values`, `ingest-config` — the filter-resolution loop |
| `skills/retrieval/dashboard` | response sections incl. `s3_inventory` and `pipeline_alignment`; how to diagnose un-indexed segments |
| `skills/retrieval/suggest-prompts` | generated example queries / notable recent events |
| `skills/retrieval/vastdb-read` | direct `vastdb` SDK over SSH tunnel; `list_catalog.py` |
| `skills/ingest/reingest-videos` | whole video/stream re-ingest, with the full multi-step interaction contract |
| `skills/ingest/reingest-chunk` | one Explore chunk, incl. filename → `original_video` URI completion |
| `skills/ingest/upload-video` | adding a *new* local file via `POST /api/v1/videos/upload` (not the challenge path) |
| `skills/deployment/deploy-app-no-registry` | the hackathon deploy pattern: ConfigMap + Secret + Ingress `/app` |
| `skills/deployment/deploy`, `build-yamls`, `health` | the official retrieval stack's own deploy path — not needed for a mini-app |
| `skills/gpu/README.md` | the four model endpoints with verified curl + the per-NIM health table |
| `skills/gpu/model-health`, `model-smoke-test` | liveness and minimal real inference |
| `skills/ask-cosmos` | health check + a prepared question snippet (posts nothing) |
| `skills/submission` | drafts `SUBMISSION.md` from your code |

## What the skills imply about the backend (didn't need a screenshot)

- Namespace and trust boundary: **your team only** (`$USERNAME`), same host as the VSS UI.
- The Explore endpoint only returns **fully-indexed** parents — so anything listed there is
  safe to re-ingest as a whole chunk.
- Re-ingest replaces rows atomically per `(original_video, segment_number)`; row counts
  should not grow.
- The VastDB row does **not** persist the original `scenario`/`custom_prompt` — you can't
  read back the prompt that produced current captions.
- `vectors` must be 256-dim (Cosmos Embed1) or the collection needs recreating; the vector
  columns are excluded from `/metadata/schema` and are awkward to select via the SDK.
