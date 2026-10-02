# How to approach the VAST Builders Challenge

The stack is given; the day is spent on one decision and one app. Here's the order that
matters, and the traps.

---

## Step 0 — accept the shape of the problem

You are not building a video pipeline. You are building **something on top of a search
index that is already full**. Three consequences:

1. Your app's ceiling is set by **what the captions say**, not by your UI.
2. The backend is one REST surface (`/api/v1/*`) behind a JWT. There is no SDK to learn and
   no infra to stand up.
3. "Ships today" means **on-cluster + a URL** (`http://video-lab-team-<N>.cosmos.vastdata.com/app`).
   A local Flask server is not a deliverable.

## Step 1 — pick the use case from the corpus, not from imagination

The footage is traffic / driving / streets / warehouse / indoor-facility. A use case that
needs footage you don't have is dead on arrival. Map the idea to a pack:

| If the idea is about… | Pack | `camera_id` |
|---|---|---|
| highway corridor, lane behaviour, congestion | A | `i24_cam-1` |
| road safety, pedestrians near curb, brake events | B | `pie_cam-3` |
| forklifts, aisles, PPE, near-miss | C | `sdg_warehouse_cam-2` |
| street vehicle activity over a day | D | `neighborhood_cam-1` |
| urban intersections, multi-view of one moment | E | `sf_streets_cam-1…4` (verify indexed) |
| occupancy, people flow, indoor ops | F | `smartspace_cam-1` |

Pick **one** pack as the spine (A or C are highest signal per clip). Cross-pack queries are
the demo flourish, not the foundation.

Make it specific enough to be checkable: "flag a person within ~2 m of a moving forklift in
an aisle" beats "warehouse safety".

## Step 2 — decide the output shape before writing code

Two different products live on the same index. Choose deliberately:

- **Search** (`POST /search`) → ranked moments with timestamps + scores. Good when a human
  will look at the clip.
- **Answer** (`POST /agent/ask`, `/agent/search-and-answer`) → a written answer grounded in
  evidence chunks. Good when a human wants to be told.
- **Report** (`POST /videos/synthesize`) → chronological summary of one whole video.
- **Alert / action** → your own rule over detections + search results, judged by W&B
  inference. This is the "does something" tier and is where a demo stands out, because
  most teams stop at search.

Wire the endpoints in that order of dependency: login → search/agent → videos/playback →
(your logic) → deploy.

## Step 3 — run the loop by hand once, before any re-ingest

On the VM, in this order:

1. `login` → get a JWT. Cache it, re-login on 401.
2. `dashboard` → what's actually indexed, per camera, and whether ingest is healthy
   (`pipeline_alignment.pending_index`, `recent_videos[].unique_segments` vs
   `expected_segments`). **Do not trust the corpus doc; trust this.**
3. `list-metadata` → the real filterable fields and their real values (`schema` + `values`,
   not guessed column names).
4. `search` for the thing your idea needs → read the `reasoning_content` that comes back.
5. `videos` → play one hit, confirm the timestamp matches what you meant.
6. Now search for the thing your idea needs that the captions *probably* don't say
   (counts, what someone is carrying, whether a vehicle stopped).

**Step 6 is the real test.** Empty ≠ broken; it means the ingestion prompt never asked.

## Step 4 — re-ingest only what the gap requires

- Gate: the gap must be worth minutes of re-ingest **and** the target must be small
  (one chunk, or the latest N complete chunks of one stream). Don't re-ingest hours.
- 1–2 people own re-ingest; everyone else keeps building against the existing index.
- Choose prompt mode explicitly: keep original / scenario preset / custom prompt (≤800
  chars). Custom prompt overrides scenario. Blank metadata fields mean *preserve*, not
  erase. Omitted `scenario` **without** `custom_prompt` deliberately removes the old custom
  prompt.
- Poll `GET /dashboard/reingest/<job_id>` every ~4 s and report
  `completed_chunks/total_chunks`, `indexed_segments/total_segments`.
- After it completes, re-run the exact search that failed. That comparison is the demo's
  best story beat: *the gap between what you asked and what the prompt asked about.*

## Step 5 — build the app, then deploy it on the cluster

- Shape: small `main.py` (+ optional `requirements.txt`) that logs in with creds from env,
  calls the VSS API, and serves `/` and `/health` on `0.0.0.0:$PORT`. Ingress rewrites
  `/app` → `/`, so don't bake a prefix into your routes.
- Deploy path (no Docker/registry): **ConfigMap** (code) + **Secret** (VSS creds) +
  Deployment (`python:3.12-slim`) + Service + **Ingress `path: /app`** on the *existing*
  team host. Never invent a hostname, never use `/`.
- Iterate: recreate ConfigMap → `kubectl rollout restart deploy/<name>`.
- Your app's own reasoning (classify a hit, draft a summary, decide an action) goes to
  **W&B serverless inference** with the `WANDB_*` env vars — not to the VSS pipeline.

## Step 6 — ship the story

The judged artifact is: *problem → one video archive pack → search/filters across cameras →
insight or action*, plus a working URL and a repo link. Run the `submission` skill early
enough that writing `SUBMISSION.md` isn't the last 20 minutes.

---

## Where an edge is available

Ranked by leverage, from what's actually in this stack:

1. **Canary-1B ASR is unwired.** Transcripts / spoken-word search / translated subtitles
   exist as a capability nobody's pipeline uses. Highest novelty per unit of work.
2. **Action, not retrieval.** Detection metadata (`/videos/detections`, object counts in
   `dashboard.objects`) composed with search hits into a rule that fires — near-miss
   detection, dwell-time in a zone, "person + moving vehicle" co-occurrence across sites.
3. **Cross-camera/cross-site stories.** The one-index-many-lenses design is the platform's
   own thesis; a board that groups one event across I-24, Toronto, neighborhood, SF and
   warehouse cameras shows it off better than another search box.
4. **Direct VastDB reads** (`vastdb-read`) for aggregate questions the API doesn't answer
   (distributions, joins across video + prompts tables).
5. **Cosmos3-Reason called directly** for a second-pass, different-prompt description of a
   clip you already have — without touching DataEngine.

---

## Traps (each of these has burned someone in the upstream docs)

| Trap | Reality |
|---|---|
| "Let me re-ingest everything with the perfect prompt" | Minutes-to-hours of wall clock, and dedup means you can't keep both versions. Gate it. |
| Searching before verifying filter values | `metadata_filters` with a wrong value silently returns nothing. Resolve via `/metadata/values`. |
| Guessing column names | Only `/metadata/schema` fields are filterable; `tags`, `source`, `reasoning_content` are excluded. Tags go in the top-level `tags` array. |
| Using the header for playback | `/videos/stream` and `/playback-url` take the JWT as `?token=`, because `<video>` can't send headers. |
| Trusting "search found nothing" as a bug | Almost always a prompt-coverage gap. |
| Building a local server as the demo | Explicitly disallowed. K8s + `/app` only. |
| Native desktop/mobile app | Cannot be demoed or judged on the VM. Web app. |
| Ingesting YouTube | Licensing. Don't. |
| Everyone re-ingesting at once | Organizers call it out; designate 1–2. |
| `/config` in the repo | Credentials live at absolute `/config/` on the VM. Never commit, never echo. |
