# Pitfalls Research

**Domain:** Web-based 3D garment visualization / virtual fitting (photo-to-3D + react-three-fiber)
**Researched:** 2026-09-28
**Confidence:** MEDIUM (core API behavior and rendering pitfalls verified against multiple sources; cost/queue figures are approximate and must be re-verified when API keys are created)

## Critical Pitfalls

### Pitfall 1: Photo-to-3D APIs return garbage meshes for garments

**What goes wrong:**
Meshy/Tripo-class generators produce meshes that are: (a) **non-watertight** with holes and non-manifold edges — one source estimates ~70% of raw AI-generated outputs contain non-manifold edges or inverted normals; (b) **dense triangle soup** with no deformation-friendly topology (critical, since this app must skin/deform the garment on a mannequin); (c) occasionally **hallucinated geometry** — buttons become blobs, straps merge into the body, thin straps and lace disappear or get inflated into solid masses. Garments are among the hardest inputs: thin shells, concave silhouettes, semi-transparent fabrics, and repetitive patterns all degrade reconstruction.

**Why it happens:**
These APIs are general-purpose asset generators trained on solid objects. They have no garment priors, no notion of "shell with an opening," and no guarantee of manifold output. Developers demo with a sneaker or a chair, see a great result, and assume garment inputs behave the same.

**How to avoid:**
- Treat API output as **untrusted input to a validation gate**, never as final product. After download, run automated checks with `gltf-transform inspect` (or a mesh-analysis library): triangle count bounds, bounding-box aspect sanity vs. category (a t-shirt should be wider than tall), texture resolution, connected-component count.
- Preprocess inputs before submission: clean background-removed images (Pitfall 8), guidance photos flat-laid with limbs/straps not folded under.
- Budget for a **retry-with-different-parameters path** (e.g., Tripo vs Meshy as A/B, or art-style/quality presets) and a **"this garment didn't work, try another photo" UX** — not a spin-forever state.
- Consider asking the API for its quad-remesh option (Meshy offers Remesh/Retopology) if available — dense triangles deform badly in the pseudo-fit binding step.
- Track per-generation quality offline: keep a small eval set of 10–20 garment photos (white tee, striped dress, patterned shirt, thin-strap top) and re-run it whenever you change API version or parameters.

**Warning signs:**
- Demo garments look fine but "random" user garments fail; failures correlate with thin straps, patterns, or white-on-white photos.
- The viewer crashes or shows black/hole artifacts on some generated models (inverted normals, non-manifold geometry).
- Vertex counts vary wildly (50k–2M+) between generations with no attempt to control it.

**Phase to address:**
Upload + 3D-gen worker phase — the validation gate and retry path must exist the moment the first real generation is wired up, not after polish.

---

### Pitfall 2: Generated meshes are not true-to-size — scale is arbitrary

**What goes wrong:**
3D-gen APIs return meshes in **arbitrary units with no relationship to real garment dimensions**. Meshy/Tripo outputs are normalized (typically normalized into a unit-ish box), and there is no parameter that says "this is a size M t-shirt." If you load the .glb and place a mannequin next to it, the garment might be doll-sized or giant. Users judge fit visually — a garment at 70% scale on the correct mannequin reads as "this runs small," which destroys the core value proposition.

**Why it happens:**
Developers assume "GLB = meters" and that generation preserves the photo's implied scale. It doesn't — photo-to-3D normalizes geometry. Nothing in the API response carries real-world dimensions; only the LLM size extraction (product description → chest/waist/length in cm) knows the truth.

**How to avoid:**
- Make scale calibration an **explicit, deterministic pipeline step**: after download, compute the mesh's bounding box, then uniformly scale so a known dimension (e.g., garment length or chest width, from the LLM-extracted measurements) matches the extracted cm value. One canonical approach: normalize every garment into centimeter space where 1 unit = 1 cm, and build the mannequin in the same space.
- Decide a **canonical dimension policy** per category: length is most reliably extracted from descriptions ("60cm length"); use chest width as a secondary. Log both and which one was used, so regressions are diagnosable.
- Never trust the mesh's own proportions for anything except shape — trust measurements only from the description/size chart.
- Add a debug/dev mode that renders a 10cm reference grid and the mannequin's measurement landmarks so scale bugs are visible immediately.

**Warning signs:**
- The same garment "fits" a plus-size mannequin and a slim mannequin identically (scale step missing entirely).
- Fit feedback says "looks tiny/huge" despite correct mannequin body.
- Tests pass because they compare mesh-to-mesh, not mesh-to-mannequin.

**Phase to address:**
LLM size extraction phase — the measurement model and canonical units must be defined before body morphs/binding consume them.

---

### Pitfall 3: .glb assets melt mobile GPUs

**What goes wrong:**
Raw 3D-gen output is a multi-million-triangle mesh with 2K–4K textures, served as a 30–80MB .glb. On desktop it "just works." On mid-range phones: multi-second parse jank (JSON GLTF is parsed on the main thread), texture-memory blowups (mobile GPUs have far tighter texture budgets; a 4K texture is 64MB VRAM uncompressed), thermal throttling, and single-digit FPS. Texture memory, not geometry, is usually the mobile bottleneck.

**Why it happens:**
The API returns what it returns and the developer serves it directly. r3f/drei's `useGLTF` loads whatever bytes arrive; nothing downstream optimizes automatically. Also common: enabling the Draco path when the asset is meshopt-compressed (or vice versa), or relying on Google's CDN for decoder WASM (flaky in some regions, blocked by some privacy extensions).

**How to avoid:**
- Build an **asset-processing pipeline step in the worker**, not client-side: after generation, run `gltf-transform` — `prune`, `dedup`, `weld`, resize textures to ≤1024px (garment textures don't need 4K on a phone screen), compress geometry with **meshopt** (decoder is ~1.3MB total and faster than Draco's multi-WASM bundle — prefer it for mobile), compress textures to **KTX2/ETC1S** or WebP.
- Target a **delivery budget**: e.g., ≤150k triangles, ≤8MB .glb per garment. Enforce in the worker; reject/re-process assets over budget.
- In r3f: cap `dpr={[1, 1.5]}` on the Canvas, use `frameloop="demand"` (viewer is orbit-only — no need to render 60fps when idle), lazy-load the Canvas so it doesn't block page JS, and show a poster/loading state via `useProgress`.
- Set up the loader decoders explicitly and self-host decoder WASM: `useGLTF(url, true)` for Draco; import `MeshoptDecoder` from `three-stdlib` for meshopt. Match the decoder to the pipeline output — mismatched config is a silent failure (model loads but renders corrupted).
- Test on a real mid-range Android device from Phase 1, not just a desktop browser's devtools device mode.

**Warning signs:**
- .glb files in the library storage are 20MB+ and nobody noticed.
- Desktop-only testing during development; first phone test happens at "polish" phase.
- Orbiting feels fine but the page is unresponsive for 1–2s when the model arrives (main-thread parse).

**Phase to address:**
Viewer skeleton phase sets the budget and pipeline contract; 3D-gen worker phase implements the gltf-transform step.

---

### Pitfall 4: Garment/mannequin clipping and z-fighting in pseudo-fit

**What goes wrong:**
Two related failure modes when binding the garment to the morphed mannequin:
- **Z-fighting/shimmer:** where garment surface sits nearly coplanar with the body surface (tight-fit areas, and everywhere on large body presets), depth-buffer precision produces flickering, crawling artifacts that look broken.
- **Clipping/penetration:** on plus-size presets the body inflates *through* the garment — arms poke through sleeves, chest through the front panel — unless the collision push-out is per-vertex and robust. Also, naive uniform scaling distorts sleeves/collar openings (they scale thicker/thinner instead of tracking the body).

**Why it happens:**
A garment generated from photos has no notion of the mannequin's skeleton or volume. Pseudo-fit (bind vertices to nearest body surface + push-out) is inherently approximate. Developers also leave `camera.near` at default tiny values, which wrecks depth-buffer precision and amplifies z-fighting.

**How to avoid:**
- **Inflate the garment mesh outward along vertex normals by 1–3mm** as a preprocessing step so it never sits exactly coplanar with the skin — the single most robust anti-z-fighting fix.
- Use `material.polygonOffset` (factor/units) on the garment material as a fallback for remaining near-coplanar regions.
- Tune depth precision: set `camera.near` as large as usable (e.g., 0.1–0.5 in cm-space terms) and keep near/far ratio small; z-fighting worsens with a large ratio.
- For morphs: morph the garment *with* the body (garment vertices bound to body vertices before morphing, moving together), rather than morphing the body and pushing the garment out afterward — the former keeps sleeves/collars coherent, the latter lets the body escape the garment.
- Accept and communicate limits: extreme presets will hug like spandex (already agreed in PROJECT.md). Cap slider range at the point where artifacts become obvious rather than chasing physics.
- Set explicit render order / material flags so semi-transparent straps or lace don't cause sort-order flicker.

**Warning signs:**
- Artifacts only at certain camera angles (classic z-fighting signature).
- "Works on slim, broken on plus" reports — binding/offset insufficient for large deltas.
- Flickering between garment and body on tight areas after any camera movement.

**Phase to address:**
Body morphs + binding phase — this *is* the hard problem of that phase; budget it as the phase's risk, with the inflation + co-morph approach decided up front.

---

### Pitfall 5: Async job handling — lost jobs, stale polling, double-billing on retries

**What goes wrong:**
Generation takes minutes; the API is credit-billed and asynchronous. Classic failure cluster:
- **Lost jobs:** server creates the remote task, then the request times out / the process restarts before the task ID is persisted → the user sees "processing" forever while credits burn.
- **Webhook/poll drift:** you poll the task but the webhook (or vice versa) also fires → duplicate completion handling, or neither fires and nothing reconciles.
- **Double-billing on retry:** a generation "fails" client-side (timeout, tab closed), the user (or your auto-retry) resubmits, and now you pay twice for one garment; or your job queue retries without an idempotency guard.
- **Terminal-state mishandling:** API job genuinely fails/expires (some providers expire task records after N days) and your DB row stays "processing" forever.

**Why it happens:**
Developers wire the happy path (submit → poll → done) in a single request cycle. But this app's contract is: a job must survive server restarts, client disconnects, webhook replay, and user retries. Tripo's own docs call out the correct pattern explicitly: queue submissions, persist task IDs, and process webhook events **idempotently**; long jobs must never block public endpoints.

**How to avoid:**
- **Persist before submitting:** write a job row (status=queued) *before* calling the API; store the provider task ID immediately upon receipt. A startup reconciliation sweep re-polls any job stuck in queued/processing older than X minutes.
- **Webhook + polling reconciliation:** webhooks are the fast path; a periodic sweeper polls tasks that haven't reached a terminal state. Both paths write through one idempotent state transition (`UPDATE ... WHERE status='processing'` style guard, or an events table with dedup on provider task ID + status).
- **Idempotency at the user level:** a user can have at most one in-flight generation per garment slot; retries reuse the existing provider task, never resubmit. Dedupe on a client-generated request ID too, so a double-click doesn't create two jobs.
- **Distinguish failure classes:** provider-side failure (refund-eligible? credits usually are NOT auto-refunded — check each provider's policy) vs. your infra failure vs. garbage-output failure (Pitfall 1). Only the last should trigger automatic resubmission, and with a cap (e.g., max 2 attempts).
- Show the user an honest status timeline (queued → generating → finalizing) with elapsed time, plus a "this is taking too long / report problem" escape hatch.

**Warning signs:**
- Any `await` on a Meshy/Tripo call inside a request handler (blocking pattern = the root cause).
- Jobs in the DB stuck in "processing" > 30 minutes with no sweeper to catch them.
- Credit balance drops faster than successful garment count × price (double-billing signature).
- Webhook handler and poller both mutate job state with no guard (check logs for double completion).

**Phase to address:**
Upload + 3D-gen worker phase — the job state machine, sweeper, and idempotency guards are part of the worker's first version, not hardening for later.

---

### Pitfall 6: Quota/abuse enforcement missing before public launch

**What goes wrong:**
Each generation costs real money (~$0.20–0.40/garment). Without enforcement: (a) authenticated users churn through free generations via multiple accounts (email aliases are free — the app's own Gmail verification makes signup frictionless); (b) scripted abuse hits the public share-link surface or the submit endpoint directly with harvested photos; (c) one viral moment = a four-figure API bill overnight. Portkey and similar gateways added duplicate-billing protection windows for exactly this provider class — the risk is well-known.

**Why it happens:**
Quota is a product decision that feels like it can wait ("we'll add payments in v2 anyway"), and during development nobody hits limits because the team is 3 people with free-tier credits.

**How to avoid:**
- Enforce quota **server-side at job creation**, atomically (conditional increment or DB constraint) — never client-side, never post-generation.
- Quota keys: per-account is the baseline, but add **per-IP and per-device rate limits** on the submit endpoint (e.g., N generations/hour) to blunt multi-account farming, plus Cloudflare Turnstile (free) on signup and submit.
- Set a **global daily spend ceiling** (circuit breaker): if aggregate generations/day exceed a dollar cap, new jobs queue as "capacity reached" instead of submitting. This is the single control that makes public launch survivable.
- Track credit spend per job from the API response (Tripo returns credit usage in task responses) so accounting matches the provider's bill — discrepancies surface double-billing (Pitfall 5).
- Cap uploaded payload size/count server-side (image dimensions, MB limits) — also protects the 3D-gen input quality.

**Warning signs:**
- The submit endpoint has no rate limit in the code review and nobody flags it.
- Quota is checked in a React component.
- No dashboard/alert for daily generation spend; you'd only notice at the monthly bill.

**Phase to address:**
Must exist in the 3D-gen worker phase (server-side quota at job creation) with the global circuit breaker verified **before** the deployment/public-launch phase.

---

### Pitfall 7: URL scraping is fragile — fashion retail sites actively block bots

**What goes wrong:**
"Paste a store URL and we fetch images + description" collides with reality: most major retailers (Amazon, SHEIN, Zara, Nike, H&M) run Cloudflare, Akamai, DataDome, PerimeterX, or AWS WAF. Ban rates for naive scraping reach 70–90%; SHEIN is documented among the hardest targets (only 4 of 11 tested scraping APIs achieved 80%+ success on protected sites in Proxyway testing). Blocklists escalate: what works in dev from your laptop fails in production from a datacenter IP within days. The feature silently degrades to "works for some sites, some of the time."

**Why it happens:**
The feature is demoed on a personal blog or a small store with no protection. The scale of anti-bot deployment across fashion retail is underestimated, and it's treated as a solved problem ("just fetch the HTML") rather than an arms race.

**How to avoid:**
- **Design upload-first, URL-second.** Guided photo upload must be the primary, fully-functional path (it already is per PROJECT.md); URL import is a convenience that must fail gracefully ("we couldn't fetch this page — upload photos instead"), never a dead end.
- For URL fetches, use a **managed scraping/unblocker API** (Apify Web Unblocker, Zenrows, Scrapfly, Oxylabs — these exist precisely because DIY is a money pit) or retailer-specific structured APIs for the big ones (Amazon-class providers). Do not hand-roll parser + proxy rotation for v1.
- Assume **every site's DOM changes**: write per-site extraction defensively (JSON-LD `Product` schema first — many stores emit it; CSS-selector fallbacks second), and never let one broken site parser break the endpoint.
- Extract images from JSON-LD/og:image meta tags where possible — more stable than gallery DOM.
- Handle the legal/ToS dimension: fetching public product pages via an unblocker service sits in a gray zone for some retailers; a terms-of-service note and takedown responsiveness are cheap insurance.

**Warning signs:**
- A hand-written `fetch(url)` + regex/cheerio parser in the codebase with no unblocker layer.
- Success-rate metric per domain isn't tracked — you can't see the decay happening.
- Any single retailer's redesign causes a production incident.

**Phase to address:**
URL import slice of the upload phase — decide the unblocker-vs-DIY posture then, and ship the graceful-failure UX with the feature, not after.

---

### Pitfall 8: Background removal fails on the exact photos users will give you

**What goes wrong:**
The 3D-gen pipeline needs clean garment-on-transparent/neutral images. Users supply: white shirts on white backgrounds (segmentation erases the garment), patterned/busc backgrounds where the pattern merges with the garment edge, garments photographed on models (body segments get kept or the garment gets cut where skin shows), and screenshots with watermarks/UI chrome. Heuristic/threshold-based removal fails outright here; even good matting models (BiRefNet-class notably outperforms older rembg/U2Net-class models) produce halos, eaten straps, and ragged edges that then poison the 3D reconstruction.

**Why it happens:**
Background removal is tested on perfect studio flat-lays (the developer's own test images) rather than the noisy distribution of real user photos. White-on-white and low-contrast edges are known-hard cases that don't show up until real users arrive.

**How to avoid:**
- Use a **strong modern matting model** (BiRefNet-class or a commercial API) — not threshold heuristics — and run it as a **mandatory preprocessing step** before the 3D-gen submission, feeding the masked image (not the original) to the API.
- Add a **pre-flight image quality gate**: after matting, check mask coverage (garment should occupy a plausible fraction of frame; 2% or 95% coverage = bad crop or failed matting), edge quality heuristics, and minimum resolution. Reject with specific, instructive error messages ("garment blends into the background — try a photo on a contrasting surface").
- Show the user the **cutout result for confirmation before spending credits** on generation — a 2-second human check catches most matting failures at zero cost. This is the highest-leverage prevention in the whole pipeline.
- Keep a matting failure gallery: every rejected/edited photo feeds the test set.

**Warning signs:**
- Matting runs only client-side or not at all before submission.
- Test photos are all dark garments on dark tables.
- Users report "the 3D model looks melted" — usually a matting artifact upstream.

**Phase to address:**
Upload phase — quality gate + user confirmation of the cutout ships with the upload flow.

---

### Pitfall 9: Cold-start economics of 3D-gen APIs burn the free tier and the calendar

**What goes wrong:**
Two cold-start costs compound: (a) **development burn** — every iteration cycle (change preprocessing, retry, test the viewer) can silently consume generation credits; Meshy-class free tiers are ~200 credits/month with ~15 credits per generation — roughly a dozen dev generations/month before paying, which is fewer than one afternoon of iteration; (b) **queue latency** — generations take minutes and can queue at provider peak times; dev loops and user expectations both assume faster.

**Why it happens:**
Nobody tracks credits per environment; a `npm run test:e2e` that generates 5 garments per run eats the month. And the async-minutes reality isn't felt until the job system exists, so early phases over-assume synchronous dev ergonomics.

**How to avoid:**
- **Cache generations aggressively at every level:** identical input image hash → same output asset, forever. Dev/staging environments get a **recorded-fixture mode** (check in 5–10 real generated .glb outputs and their metadata; the pipeline replays them instead of calling the API). Tests never hit the real API; only an explicit, manual, flagged "live smoke test" does.
- Environment-separated API keys with different spend ceilings; dev key capped hard.
- Design the job system (Pitfall 5) so queue-wait is a first-class UI state from the start — minutes-long waits with a good status UX feel fine; minutes-long waits with a frozen spinner feel broken.
- Have a **fallback provider** wired early (both Meshy and Tripo have compatible async-task shapes): provider outage or pricing change shouldn't halt the product. Abstract the provider interface on day one — it's cheap now, expensive later.
- Verify current pricing/credit policies **when creating the accounts** (PROJECT.md flags this) — figures in this document are approximate and both vendors change pricing.

**Warning signs:**
- E2E tests that call the live API.
- No input-hash cache; resubmitting the same photo costs again.
- Single hardcoded provider client imported directly into the route handler.

**Phase to address:**
Upload + 3D-gen worker phase — fixture mode, caching, and the provider abstraction are scaffolding requirements, built alongside the first real integration.

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Serving API .glb directly with no gltf-transform step | Zero pipeline code | Mobile unusable; storage bloat; every asset re-optimized later by hand | Never — pipeline is cheap and mobile is a primary platform |
| Polling-only job status (no webhooks) | Simpler first version | Slower UX, more provider API calls; acceptable only *with* the reconciliation sweeper | Acceptable for v1 if the sweeper exists |
| Trusting mesh bounding box as "true size" | Skips measurement plumbing | Core value prop (true-to-size) is quietly wrong | Never |
| Hardcoding one 3D-gen provider client | Faster first integration | Provider outage/price change = full stop; refactor touches every call site | Never — thin provider interface is ~a day of work |
| Client-side quota checks only | No DB work | Trivially bypassed; direct API cost exposure | Never |
| Skipping mesh validation gate | Ship faster | Garbage models reach users; support burden; trust damage | Only during the very first hello-world integration, removed within the same phase |
| Per-site hand-written scrapers without JSON-LD-first strategy | Quick wins on 2–3 sites | Maintenance treadmill as sites redesign | Acceptable only for the top 3 retailers, layered over a generic fallback |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Meshy API | Assuming output is real-world scale/watertight; treating generation failures as rare | Validate every output (Pitfall 1); expect non-watertight as the norm; check credit policy on failed tasks when creating the account |
| Tripo API | Ignoring credit-usage field in task responses; non-idempotent webhook handling | Log credit usage per job for spend accounting; dedupe webhook events on task ID + status |
| Both providers | Blocking request handler on generation; no reconciliation when task record expires | Async worker + persisted task IDs + startup sweep; handle provider-side task expiry explicitly |
| r3f / drei useGLTF | Mismatched decoder (Draco vs meshopt) vs. actual asset compression; Google-CDN decoder dependency | Match decoder setup to pipeline output exactly; self-host decoder WASM |
| Next.js + Canvas | Canvas bundles into the initial JS payload; no Suspense boundary | `next/dynamic` the Canvas with `ssr: false`, poster frame while loading |
| Scraping/unblocker | Treating HTML parsing as stable; one parser per site with no observability | JSON-LD first, tracked per-domain success rate, generic fallback |
| LLM size extraction | Free-form text response; trusting one dimension; no region normalization | Structured output (JSON schema) + validation ranges + region → cm normalization (already flagged in PROJECT.md) |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Uncapped devicePixelRatio on Canvas | Blurry-or-laggy dichotomy on phones; thermal throttling | `dpr={[1, 1.5]}` | Immediately on 3x-DPR phones |
| Continuous render loop for an orbit-only viewer | Battery drain, jank on low-end devices | `frameloop="demand"` + invalidate on controls change | First real phone test |
| 4K textures from generation pipeline | 64MB+ VRAM per garment; texture thrashing | Resize to ≤1024 + KTX2 in worker pipeline | First mid-range Android test |
| Full library page loading all garment GLBs at once | Library page freezes; memory balloons | Lazy-load per viewer instance; thumbnails as posters | ~5+ saved garments |
| No asset disposal on route change | GPU memory leak; tab crash after browsing several garments | Dispose geometries/textures on unmount (or rely on r3f auto-dispose correctly — manual `useEffect` holds defeat it) | Multi-garment browsing sessions |
| Main-thread GLTF parse of large files | 1–3s page freeze on model arrival | Meshopt compression (fast decode) + keep triangle budget; loading poster | >10MB .glb on mobile |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Public share links exposing arbitrary user-uploaded content with no moderation path | Share links used to host inappropriate images via your domain | Report/abuse link on share pages; basic image checks on upload; kill-switch to disable a share link |
| Submit endpoint accepts arbitrary URLs → SSRF | Internal network probing via the scraper | Validate URL scheme/host against public IPs only; never fetch internal ranges; timeout + size caps |
| Provider API keys in client-accessible env vars (`NEXT_PUBLIC_`) | Direct credit theft — attackers generate on your dime | Keys server-side only; all provider calls from the worker |
| Unlimited image upload size | Storage/DoS cost, and giant images waste generation credits | Server-side dimension + byte caps before matting/submission |
| Unauthenticated job-status polling by job ID | Enumeration of other users' generation activity | Job status endpoints scoped to owning account (share links are the only public surface, read-only by design) |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Frozen spinner during multi-minute generation | Users assume broken; tab abandonment; duplicate submissions | Staged status (queued → generating → finalizing) with elapsed time + in-app notification when done (per PROJECT.md) |
| Garment silently fails generation with generic "error" | Users blame themselves or the product; churn | Specific failure messages keyed to failure class (bad photo, busy, unsupported garment) + retry path |
| No cutout confirmation before spending credits | Users pay (quota) for a melted model caused by a bad photo | Show the background-removed result for approval pre-submission |
| Fit display ignores regional sizing on screen | Asian-L users see "L" and misjudge fit | Normalize to cm internally (per PROJECT.md) and display both original and normalized size |
| Pseudo-fit spandex effect on extreme presets presented without context | "This app is garbage, everything clings" | Cap slider range honestly; show a "fit is approximate" affordance on large deltas (already agreed in PROJECT.md) |

## "Looks Done But Isn't" Checklist

- [ ] **3D generation flow:** Often missing reconciliation for stuck jobs — verify a job stuck >30min transitions to failed and refunds quota
- [ ] **Scale correctness:** Often missing cross-check of mesh size vs. extracted measurements — verify in dev mode that a known 70cm garment measures 70cm in-scene
- [ ] **Mobile viewer:** Often missing real-device testing — verify on an actual mid-range Android at throttled network (Slow 4G), not desktop device-emulation
- [ ] **Quota:** Often missing server-side atomic decrement — verify concurrent double-submit cannot exceed quota
- [ ] **Webhook handling:** Often missing idempotency — verify replaying the same webhook twice produces one state transition
- [ ] **URL import:** Often missing graceful failure on bot-blocked sites — verify a Cloudflare-protected URL degrades to upload prompt
- [ ] **Share links:** Often missing abuse reporting + disable path — verify a share link can be killed by its owner
- [ ] **Credit accounting:** Often missing per-job spend logging — verify sum(logged spend) is reconcilable against the provider dashboard
- [ ] **Provider fallback:** Often missing — verify the provider interface has a second implementation, even unexercised

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Garbage mesh reached a user | LOW | Apologize + quota refund + feed photo to failure gallery; auto-validate gate prevents recurrence |
| Double-billing detected | MEDIUM | Reconcile job spend log vs. provider dashboard; contact provider support for credit refunds (documented policies vary); patch idempotency guard |
| Quota bypassed / spend spike | LOW if circuit breaker exists; HIGH if not | Kill-switch the global ceiling immediately; patch quota; if no breaker existed, this is the incident that justifies building it same-week |
| Scraper broken by retailer redesign | LOW | Domain flagged degraded; users fall back to upload; fix parser behind the generic fallback |
| Mobile perf unacceptable at polish phase | MEDIUM | Retro-fit gltf-transform pipeline over existing assets (batch re-process from stored originals — hence keep original API outputs in storage) |
| Z-fighting/clipping at binding phase | MEDIUM | Add normal-inflation preprocessing + polygonOffset; if pervasive, revisit co-morph approach — expensive, so decide binding strategy up front (Pitfall 4) |
| Provider outage or price shock | LOW with abstraction; HIGH without | Route to fallback provider; re-run failure gallery eval on the new provider before switching |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| 1. Garbage meshes | Upload + 3D-gen worker | Validation gate rejects a planted bad mesh; failure-gallery eval pass rate tracked |
| 2. Scale not true-to-size | LLM size extraction | Dev-mode ruler check: known-cm garment measures correctly in-scene |
| 3. Mobile GPU perf | Viewer skeleton (budget) + worker (pipeline) | Real mid-range Android, throttled network, ≤8MB asset, stable FPS |
| 4. Clipping/z-fighting | Body morphs + binding | Visual check across full slim→plus slider range at multiple angles |
| 5. Lost jobs / double-billing | Upload + 3D-gen worker | Kill worker mid-job → recovers on restart; webhook replay is idempotent; spend log reconciles |
| 6. Quota/abuse | Worker (enforcement) + deployment (breaker) | Concurrent-submit test cannot exceed quota; global breaker halts submissions at cap |
| 7. Scraping fragility | Upload phase (URL import slice) | Protected-site URL degrades gracefully; per-domain success rate tracked |
| 8. Background removal | Upload phase | Quality gate rejects planted white-on-white photo; cutout confirmation shown pre-submission |
| 9. Cold-start burn | Upload + 3D-gen worker | E2E suite runs with zero live API calls (fixture mode); dev key spend capped |

## Sources

- [Meshy vs Tripo vs V2Fun comparison — OurCodeWorld (Aug 2026)](https://www ourcodeworld.com) — non-watertight output, scale, manual cleanup expected
- [How to 3D Print Meshy AI Models — PartFit3D](https://partfit3d.com) — mesh repair + manual rescaling required post-export
- [DIY3D blog (May 2026)](https://diy3d.blog) — ~70% of raw AI 3D outputs contain non-manifold edges/inverted normals (LOW confidence, single source)
- [ToolShub AI (Sep 2025)](https://toolshub.ai) — Meshy Remesh/quad retopology for deform-friendly topology
- [Tripo3D UGC platform guidance](https://www.tripo3d.ai) — async architecture, idempotent webhook processing, queue + persisted task IDs
- [Tripo OpenAPI changelog — docs.tripo3d.ai](https://docs.tripo3d.ai) — credit usage in task responses; API billing plan docs
- [Portkey Enterprise Gateway docs (Sep 2026)](https://docs.portkey.ai) — 24h duplicate-billing protection for Meshy/Tripo credit spend (evidence the double-billing risk is real)
- [Meshy credit pricing (community-reported)](https://www.reddit.com) — ~15 credits/generation, ~200 free credits/month (MEDIUM confidence, verify at account creation)
- [Interface Lab runtime pipeline guide](https://www.interfacelab.design) — gltf-transform pipeline: prune/dedup/resize, meshopt, KTX2, CI-enforced
- [Three.js performance fixes — Hon Tran](https://www.hontran.dev) — draw calls, dpr, render-on-demand, disposal, renderer.info
- [motionprompts.dev pipeline notes](https://motionprompts.dev) — meshopt decoder ~1.3MB vs Draco bundle; faster decode on mobile
- [three.js discourse: polygonOffset and z-fighting](https://discourse.threejs.org) + [three.js issue #2593](https://github.com/mrdoob/three.js/issues/2593) — polygonOffset mechanics and limitations
- [a327ex 3D exploration log](https://a327ex.com) — garment/body coplanar z-fighting case study
- [three.js GitHub: SkinnedMesh.bind as armature](https://github.com/mrdoob/three.js) — binding garments to skeletons, bindposes
- [Proxycove: reducing scraping ban rates (Feb 2026)](https://proxycove.com) — 70–90% block rates without countermeasures; protection stack landscape
- [Proxyway scraping API testing (via LinkedIn)](https://www.linkedin.com) — only 4/11 scraping APIs exceeded 80% success on protected sites; SHEIN among hardest
- [OpenWeb Ninja e-commerce API comparison (Sep 2026)](https://www.openwebninja.com) — structured retailer APIs as alternative to scraping
- [Razzzila/Nikita Isakov batch background removal notes](https://razzzila.nikitaisakov.com) — heuristic thresholding fails on low-contrast product/backdrop pairs
- [Noise/oto.net AI Week](https://noise.getoto.net) — BiRefNet adoption for background removal quality

---
*Pitfalls research for: web-based 3D garment visualization / virtual fitting*
*Researched: 2026-09-28*
