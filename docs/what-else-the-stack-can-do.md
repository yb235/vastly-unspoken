# What else the stack can do

Beyond "search → review board", what capabilities are actually lying unused? Written against the
verified capability map (`docs/skills-capability-map.md`), not the marketing blurb.

---

## The primitives people under-use

Most teams will use primitives 1, 5 and 6. The interesting products come from 2, 3, 4.

| # | Primitive | What it really unlocks |
|---|---|---|
| 1 | **Hybrid retrieval** (`POST /search`) | Find candidate moments by text. Everyone uses this. |
| 2 | **On-demand inference on an arbitrary clip** (Cosmos3-Reason `video_url` base64, YOLO `/v1/infer` base64) | **Facts the index never contained** — counts, motion state, proximity bands, structured JSON — computed at request time, *without* mutating the shared index. This is the fix for the caption-coverage problem, and it is not re-ingest. |
| 3 | **Raw VastDB reads** (`vastdb-read`, SSH tunnel + SDK) | Whole-corpus aggregation: per-camera census, per-day activity, object-class distributions, joins against the prompts table. |
| 4 | **W&B serverless inference + run/table logging** | The app's **only** persistence (no write path exists) *and* the measurement harness. Two jobs, both real. |
| 5 | **Playback + per-segment evidence** (`/videos/stream`, `/metadata`, `/detections`) | Grounded evidence, before/event/after, bbox crops. |
| 6 | **Deploy an arbitrary small HTTP service** at `/app` | Not only a UI — an API, a bot endpoint, a generated-artifact host. |
| 7 | **Canary-1B ASR/translation** (unwired) | Voice in, or **audio out of footage** — the only untouched modality. |

Known limits that constrain design: **no client-side vector math** (reading `vectors`/`vectors_visual`
via the SDK can fail without the pipeline's patch — so similarity work must go through `/search`,
and whether a *visual* query is accepted is unverified), **no write path** for our own data, ConfigMap
≈1 MiB, and re-ingest mutates a shared index atomically.

---

## Tier 1 — additive to the current plan (do these)

### 1. On-demand structured extraction ("the index finds it, the models measure it")
The index retrieves candidates; then for each candidate we run YOLO and Cosmos3-Reason on the clip
and extract a **structured record**: person count, vehicle count, motion state, orientation,
proximity band, occlusion, path overlap — with the model's own confidence and the exact clip hash.

Why it matters: it converts our biggest risk (captions may not contain counts or relational
language) into a product feature. Ranking and filtering then run on data the index alone cannot
provide, and **nothing in the shared index is touched**.

### 2. Retrieval quality measurement (W&B as the honest sponsor story)
Build a tiny labelled set (30 query → expected-clip pairs drawn from real results), sweep
`min_similarity` and `hybrid_text_weight`, log precision@k and latency to W&B, and ship the chart.
Then say which setting the app uses and why.

Why it matters: it makes W&B do experiment tracking (not just one LLM call), gives judges a
quantitative claim, and is the only way to answer "is this retrieval actually good?" honestly.

### 3. Chain-of-custody evidence packets
Every review packet carries a verifiable trail: SHA-256 of each clip's bytes as streamed, the model
IDs and versions used, the retrieval parameters, segment identifiers, timestamps, and the git commit
of the deployed app. Tamper-evident, printable, and reproducible.

Why it matters: it's the difference between "a nice demo" and "something an investigator could
actually put in a file". Costs almost nothing to build and no other team will do it.

### 4. Timeline / storyboard renderer for a parent video
Render one parent chunk as a navigable track: caption per segment, object classes, detection chips,
jump-to-moment, with the whole-video synthesis on top (`/tools/segments` + `/videos/synthesize`).

Why it matters: it makes the underlying data model *visible* in one screen — the clearest possible
demonstration that the index is structured rather than a pile of clip search.

## Tier 2 — alternative products, if we pivot

### 5. Query→compute analyst ("how many, how often, where")
A question like *"how many moments have a person and a forklift in the same aisle?"* is answered by
retrieve → on-demand detection → count, with every number linked to the clips that produced it.
Targets the exact class of question captions can't answer.

### 6. Camera profile / activity census
Per-camera, per-hour activity profile from raw VastDB reads + object-class counts: what each camera
sees, when it's busy, what objects dominate. Product for a person choosing which cameras to watch,
or auditing coverage.

### 7. Index trust panel (caption vs detection disagreement)
Cross-check captions against detections: caption says "empty corridor" but YOLO found a person;
caption says "two vehicles" but three boxes. Surface disagreements as an index-quality signal.

Why it matters: it is a genuinely novel angle, cheap to compute, and it makes the app *more*
credible — "we show you where the index is unsure" beats pretending captions are ground truth.

### 8. Batch sweep agent → ranked register
Run the pipeline over the retrieved candidate set in one pass (bounded: e.g. top-N hits), verify
each with on-demand inference, and emit a ranked **review register** as the day's artifact, plus a
per-cluster digest. Turns a one-shot query into an autonomous workflow with a deliverable.

## Tier 3 — cheap surprises

| Idea | Cost | Why it might land |
|---|---|---|
| **Voice → structured query** — Canary transcribes, W&B LLM parses the transcript into `query` + `time_filter` + `metadata_filters` | Low | Shows two models cooperating on the input path; the parser is where "plain language" becomes precise API calls |
| **"Same moment, several angles"** (I-24 `scene*_p*c*`, SF cams 1–4) | Medium, **unverified alignment** | Nobody else will exploit the multi-camera overlap; drop it if timestamps don't align |
| **Caption overlay store** — upgrade captions for a few clips via Cosmos3-Reason on demand, keep them in W&B/our own JSON, never mutate the shared index | Low | Demonstrates enrichment *without* re-ingesting — a real architectural alternative to the platform's expensive lever |
| **Before/after re-ingest proof** — one chunk, observation-only prompt, same query re-run | Medium | Makes DataEngine legible and shows the prompt→coverage causal link |
| **Audio probe on footage** — run Canary over a clip's audio track | Low (likely empty) | Either finds speech nobody knew was there, or becomes an honest "this corpus is visually-only" finding |
| **Spatial/visual query** — probe whether `/search` honours a visual-dominant query (`hybrid_text_weight` low, near-image phrasing) | Low | If it works, "find clips that look like this" is a capability nobody will demo |

## Tier 4 — deliberately avoid

- Rebuilding the DataEngine graph, or re-ingesting at volume (shared, expensive, and not our edge).
- Any emotion / intent / deception / fault claim; any identity or cross-camera person tracking.
- Anything that needs a live stream: the corpus is fixed footage.
- Relying on prompt-suggester "key events" (Team 24 has zero).
- Browser-native speech recognition as a Canary fallback — it would silently remove a sponsor from
  the demo.

---

## Recommendation

Keep the conflict-review product as the spine, and bolt on **Tier 1 items 1–3**: on-demand
structured extraction (fixes the caption risk and makes NVIDIA models essential), the W&B retrieval
measurement (turns W&B from decoration into evidence), and the chain-of-custody packet (the
differentiator that costs a day's worth of nothing).

Then pick **one** Tier 3 flourish by cost: the voice→structured-query parser if Canary verifies
clean, or "same moment, several angles" if cross-camera timestamps align. Both are things the
retrieval-only teams cannot copy by lunchtime.
