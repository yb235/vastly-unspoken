# Use case proposal — Vastly Unspoken

**Decision: build a voice-operated, multi-camera *conflict review* workspace for a safety analyst —
not another search box, and not a detector.**

One sentence: *an analyst asks a question by voice, gets candidate person–vehicle interactions
across every camera in the team index, inspects each one against real clips + detections + captions,
and ships a cited review packet whose accept/reject decisions feed back into a W&B log.*

Naming note: the repo/product name `vastly-unspoken` is kept. "Unspoken" is load-bearing — the
product reads **observable nonverbal cues** (body orientation, hesitation, step forward/back,
yielding, path convergence), never emotion, intent, deception, or fault.

---

## 1. Why this shape, given the actual index

Codex's read of the Team 24 dashboard (re-verify before relying on it):

| Source | Segments | Share | Supports |
|---|---:|---:|---|
| Toronto live driving (`pie_cam-3`) | 1,080 | 46% | Dashcam road scenes, pedestrians, intersections, braking |
| SF street cams (`sf_streets_cam-1…4`) | 545 | 23% | Fixed urban cameras, crossings, multiple viewpoints |
| Neighborhood (`neighborhood_cam-1`) | 307 | 13% | Day-scale street vehicle activity |
| I-24 highway (`i24_cam-1`) | 180 | 8% | Multi-camera-per-scene highway behaviour |
| Indoor smart space (`smartspace_cam-1`) | 180 | 8% | People flow, facility aisles |
| Warehouse (`sdg_warehouse_cam-2`) | 60 | 2.5% | Forklift/person aisle review |

Totals reported: 2,352 searchable segments, 414 fully-indexed parent chunks, 100% caption /
detection / object-class coverage, 48 stream sessions, zero generated key events.

Three conclusions that drive the design:

1. **Roads and streets are 87% of the evidence base.** Any product whose centre of gravity is the
   warehouse is betting on 2.5% of the index. Warehouse stays a *second environment*, not a pillar.
2. **Zero key events** means the prompt-suggester layer is not a foundation. Don't build on it.
3. **The unique asset nobody else will exploit** is multi-camera coverage of one moment (I-24
   `scene*_p*c*`, SF cams 1–4) plus day-scale merges (neighborhood, two full days). "One moment,
   several angles" and "day-scale behaviour" are differentiators the corpus actually supports —
   *if* timestamps align across cameras (unverified; see §6).

And the honest counterweight: the organizers' own anchor query ("person close to a moving vehicle")
will be *every* team's demo. Matching it is not differentiating; supporting it as the spine while
adding a capability they can't easily copy is.

## 2. The user

A **safety / operations analyst** who today watches footage by hand: fleet safety reviewing dashcam
footage, or a site safety lead reviewing shared-space movement. Their job is not "find videos" —
it's *decide which moments deserve a human follow-up, and be able to justify that decision.*

That gives the product a real action: **produce a review packet**, and **record the decision**.

## 3. The three demo cases (one story, three beats)

1. **Road-crossing negotiation** — "find moments where someone looks ready to cross while vehicles
   are still moving." Toronto + SF. Observable cues: person at curb, orientation, step forward/back,
   vehicle still moving, apparent path convergence. *(69% of the index.)*
2. **Shared-aisle review** — "where do a person and an industrial vehicle share an aisle." Warehouse
   + indoor. Narrower, commercially concrete, thin evidence (240 segments combined) → the short beat.
3. **Cross-site convergence** — one question across all sites, grouped by location/camera/environment.
   This is the VAST platform story and the organizers' own cross-group payoff.

**Differentiator layer (stretch, verified before promising):** for one candidate, show *the same
moment from multiple cameras* on I-24 / SF. If cross-camera timestamps don't align, drop the
feature — never fake synchronisation.

## 4. How each sponsor earns its place (no logo soup)

| Sponsor | Necessary job | Visible proof |
|---|---|---|
| **VAST Data** | S3 video + segments, VastDB index, VSS search API, Kubernetes namespace | Real clips, filters, timestamps, deployed `/app`, one controlled re-ingest |
| **NVIDIA Cosmos3-Reason** | Segment descriptions; optional second pass over one clip with an observable-cue prompt | Caption per result + before/after caption comparison |
| **NVIDIA Cosmos Embed1** | Hybrid retrieval that finds the moment in the first place | Similarity scores in the evidence drawer |
| **NVIDIA YOLO11** | Object classes / counts / boxes as physical evidence | Detected-object chips + bbox crops |
| **NVIDIA Canary-1B** | **Voice input**: the analyst speaks the query, Canary transcribes it | Editable transcript in the search bar (Canary is otherwise unused by the platform — this is the novelty) |
| **CoreWeave** | The GPUs all of the above run on | Real model latency + endpoint/model provenance per stage |
| **W&B by CoreWeave** | Two jobs: (a) serverless inference writes the evidence-constrained review summary; (b) **the app's system of record** for review decisions (see §7) | One W&B run/table per investigation: transcript, filters, returned segment IDs, similarity, latency, recommendation, reviewer accept/reject |
| **Cursor** | Build + operate the stack via the challenge skills | Repo history, skill-driven deploy, final stack list |

## 5. What we refuse to claim (guardrails, stated in the UI)

- YOLO detects objects; it does not infer emotion, intent, or attention. Bounding boxes do not give
  calibrated metres without camera calibration. A single frame cannot establish motion.
- Cosmos captions can be incomplete or wrong. The app surfaces **review candidates**, not verified
  incidents or violations.
- No identity recognition, no cross-camera person tracking, no employment/legal/insurance decisions.
- **Human review is the final step**, and the UI says so.

Language discipline: "candidate interaction", "for human review", "the video does not establish…".
Never "near miss", "unsafe", or "at fault" as an assertion.

## 6. The falsification test — run this before writing product code

Everything above depends on one unproven fact: **do the existing captions contain the interaction
language this product needs?** Run this probe in the Team 24 index first (read-only, no re-ingest):

| Probe query | What we need back |
|---|---|
| `person close to a moving vehicle` | hits across ≥3 sites with proximity/interaction wording |
| `pedestrian near the curb while cars move` | person-position + vehicle-motion language |
| `pedestrian crossing while vehicle waits` | two-agent relational language, not just "a car drives" |
| `vehicle braking hard` | vehicle-motion states (for the road-safety beat) |
| `person and forklift in aisle` | warehouse proximity wording |
| `group of people walking in a corridor` | indoor sanity check |

Score each on: hit count, site spread, and **whether `reasoning_content` describes relations and
motion or merely objects and scenery.**

- **Pass (≥3 of 6 relational, multi-site):** build as scoped.
- **Partial:** re-ingest **exactly one or two chunks** with the observable-cue prompt below, verify
  the improvement, and fold that before/after into the demo (it makes DataEngine legible).
- **Fail:** pivot the spine to what the captions *do* cover richly (e.g. traffic-state and
  vehicle-motion review instead of person–vehicle conflict). Say so rather than building on sand.

Candidate re-ingest prompt (≤800 chars, deliberately observation-only):

> Describe only visible, observable human–vehicle interaction cues. For each person and vehicle,
> describe movement state, direction, body orientation, hand or arm gestures, stopping or yielding,
> possible path overlap, relative proximity, occlusion, and temporal sequence. Separate direct
> observations from possible interpretations. Do not infer identity, emotion, intent, fault, or
> mental state. Mark unclear features as uncertain and include relevant timestamps.

Second unverified item to check in the same pass: **do cross-camera timestamps align** within an
I-24 scene / across SF cams? And does the `/app` host serve **HTTPS** (browser microphone needs it)?

## 7. Architecture, and the constraint that shapes it

```
browser mic (or typed fallback)
   → Canary-1B transcription (editable)
   → VSS /agent/search-and-answer + /search  (Cosmos Embed1 → VastDB)
   → evidence: /videos/stream (clips) + /videos/detections (YOLO) + captions/timestamps
   → optional Cosmos3-Reason second pass on the selected clip
   → W&B serverless inference → constrained review object
   → W&B table: the decision log
   → Team 24 K8s /app  (ConfigMap code, Secret creds, Ingress path /app)
```

**The constraint nobody mentions:** the platform gives **no write path for our own data**. There is
no app database, no KV, no create-row API — upload/re-ingest only rewrite video rows. So:

- Reviewer decisions and generated packets **cannot** live in VastDB.
- Pod filesystem is ephemeral (ConfigMap-mounted code; restarts lose state).
- Therefore **W&B is the app's system of record** for review decisions — which happens to be exactly
  the honest way to make W&B load-bearing rather than decorative.

Other hard limits to design around: ConfigMap total ≈ 1 MiB (no bundled assets); playback takes the
JWT as `?token=`; filter values must be resolved via `/metadata/schema|values`; re-ingest mutates
the **shared** Team 24 index and is atomic per segment slot.

## 8. Scope discipline for a one-day build

**In:** one page; voice + typed query; result list grouped by site/camera; per-candidate
before/event/after clips; object chips + bbox; evidence drawer exposing each stage; review packet
markdown; accept/reject logged to W&B; `/health`; deployed at `/app`.

**Out (explicitly):** user accounts, persistence beyond W&B, alerting/email/webhooks, model
training, custom model deployment, live/streaming ingest, any claim of incident detection, and
any UI that implies the system decides rather than surfaces.

## 9. Open risks, ranked

1. **Caption coverage** (§6) — the single thing that can invalidate the concept.
2. **Cross-camera time alignment** — kills the "every angle" stretch goal, not the product.
3. **Microphone over HTTP/Ingress** — fallback is WAV upload and typed query; never silently swap in
   browser speech recognition (that would stop demonstrating Canary).
4. **Shared-index re-ingest** — one chunk, announced, verified; not at volume.
5. **Demo honesty** — a "review candidate" that turns out to be an empty street costs credibility;
   curate the three demo queries and verify each returns visually legible people *and* vehicles.
