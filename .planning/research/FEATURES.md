# Feature Research

**Domain:** Web-based 3D garment visualization / virtual fitting for online clothing shoppers
**Researched:** 2026-09-29
**Confidence:** MEDIUM (competitor landscape well-documented; specific feature-level details of closed B2B tools inferred from public materials — noted per-claim below)

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Guided photo upload (front + back) with example thumbnails | Every capture flow (Zyler, 3DLOOK, Zeekit) uses guided multi-step capture; users fail at free-form upload | LOW | Static guide overlay ("flat lay, plain background, garment fully visible") + client-side image validation; guided flow is in PROJECT.md scope |
| Progress feedback during async generation | 3D generation takes minutes (Meshy/Tripo reality); any AI product without progress UI reads as broken | LOW | Job status UI + in-app notification already in PROJECT.md scope; show indeterminate progress states, not fake percentages |
| 360° orbit + pinch/wheel zoom | Universal expectation from Sketchfab, product configurators, AR viewers; anything less feels like a static image | LOW | Three.js OrbitControls via react-three-fiber; include touch support and damping — frictionless rotation is the "wow" moment of the product |
| Load progress indicator in viewer | glTF/GLB garment + mannequin assets take seconds to load; blank canvas = bounce | LOW | useProgress from drei; skeleton/spinner until first frame |
| Multiple body type presets (including plus-size) | Zeekit shipped ~50 diverse preset models; shoppers expect to see themselves represented | LOW | Blend-shape presets (slim → plus) already decided in PROJECT.md; ship 4–6 named presets, not raw sliders only |
| Size selector (S–XXL) with visible change | Shoppers think in size labels, not measurements; seeing S vs XL rendered is the core fit question | LOW | Already in PROJECT.md; garment swaps/scales per extracted size chart |
| Saved garment library | Any tool with expensive AI outputs (Meshy, Tripo) persists generations; regenerating costs real API money | LOW | Already in PROJECT.md; grid of garment cards with thumbnail, name, date |
| Delete/rename garments | Basic library hygiene; quota-limited users especially need to manage their N slots | LOW | Trivial CRUD once library exists |
| Mobile-responsive viewer | Majority of clothing shoppers browse on phones; Walmart's "Be Your Own Model" is mobile-first | MEDIUM | react-three-fiber works on mobile but needs touch tuning, viewport sizing, and reduced-poly fallbacks |
| Public share link (read-only 360° view) | Sketchfab's core pattern — viewing shared models requires no account; virality depends on zero-friction viewing | LOW | Already in PROJECT.md; tokenized URL, viewer-only page, no auth |
| Account auth (email/password + verification) | Persistent library and quotas are impossible without accounts; users of AI tools expect a login wall before spending generations | LOW | Already in PROJECT.md |
| Free tier quota with clear remaining count | Every AI 3D generator (Tripo: 300 credits/mo, Meshy, Draw3D) gates generations; hidden limits feel like a scam, visible limits feel fair | LOW | Already in PROJECT.md; show "X of N garments left this month" prominently near the create button |
| Re-render/regenerate on failure | 3D generation APIs fail non-trivially (bad photos, Meshy/Tripo hiccups); users blame the app, not the API | MEDIUM | Detect failed jobs, offer one-click retry without consuming quota; log failure reason |

### Differentiators (Competitive Advantage)

Features that set the product apart. Not required, but valuable.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Paste-a-product-URL import (fetch images + description) | NO surveyed competitor ships this for 3D garment generation — dropshipping importers prove the pattern, but nobody applies it to 3D try-on; removes the #1 friction (finding/saving photos) | MEDIUM | Scraper per major store layout + OG-image fallback + manual photo picker from fetched candidates; fragile long-term (site changes) — treat as best-effort with upload fallback |
| LLM size extraction with regional normalization | Directly answers "will MY size fit?" — competitors (True Fit, 3DLOOK) do fit advice but require brand integrations; this works on ANY store's text | MEDIUM | Already in PROJECT.md; the differentiator is universality (any product description), not the extraction itself |
| True-to-size scale on adjustable mannequin | Zeekit/3DLOOK approximate fit in 2D; a real 3D garment at true scale on a body like yours is the product's core "wow" — no consumer tool surveyed does this from just 2 photos | HIGH | Already in PROJECT.md (pseudo-fit binding); honesty about spandex-hug at extremes is essential copy |
| Body sliders (height / chest / waist / hips) beyond presets | Presets get users 90% there; sliders close the gap between "similar body" and "my body" — 3DLOOK's whole pitch is body accuracy | MEDIUM | Blend-shape interpolation between preset extremes; phase after presets are proven |
| Comparison view (two sizes side-by-side) | The actual purchase question is "S or M?" — no surveyed consumer tool answers it visually | MEDIUM | Render both garments in one scene or quick-swap toggle; cheap once size scaling works |
| Turntable video export / shareable animation | Static share links are table stakes; auto-rotating video is natively shareable on social and drives the organic growth loop share links exist for | MEDIUM | Client-side MediaRecorder of canvas or server-side render; defer to v1.x |
| AR Quick Look / Scene Viewer export (USDZ) | Sketchfab's most-praised feature — "see it in my space"; for garments, "in my mirror" — low effort via glTF→USDZ conversion, high delight | MEDIUM | model-viewer or USDZ conversion pipeline; v2 candidate |
| Measurement overlay (garment flat measurements vs. body) | Virtusize-style "garment vs. your measurements" comparison builds trust in the LLM extraction; shows the numbers behind the 3D | LOW | Display extracted measurements alongside mannequin measurements in a small panel; pure UI once extraction works |
| Outfit multi-garment composition | Zeekit's rental trial influenced 30%+ of John Lewis rental sales via full-look dressing; top-of-funnel engagement multiplier | HIGH | Requires per-garment binding and occlusion; v3 territory, note only |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Real cloth physics simulation (drape) | "It would look so much more real" | Research-grade (Neural Tailor / Sewformer territory); months of R&D for marginal perceived gain at MVP fidelity; already excluded in PROJECT.md | Pseudo-fit vertex binding + honest messaging; revisit in v3 |
| 2D photorealistic try-on of the user's own photo (Zeekit/Zyler clone) | "Competitors do it, users ask for it" | Different tech stack entirely (diffusion models); splits focus from the 360° core value; quality bar set by Walmart is unreachable at indie cost | Keep as possible bolt-on post-v1 per PROJECT.md; do not let it enter MVP scope |
| User body measurement from photos ("scan yourself") | "Then it would fit ME exactly" | Body-scan accuracy is 3DLOOK's entire moat with years of tuning; bad scans → wrong sizes → broken trust; privacy surface area | Blend-shape presets + sliders; let users self-tune |
| Self-hosted 3D generation (TRELLIS/Hunyuan3D) in v1 | "Eliminate the $0.30/garment API cost" | GPU infra + model tuning is a second product; blocks validation of demand | Meshy/Tripo API per PROJECT.md; revisit after v1 |
| In-viewer fabric/material editor (change color, texture) | "It's a cool configurator feature" | Garment is generated from real product photos — recoloring fabric breaks the "real garment" promise and the true-to-size pitch; texture editing is a different product (design tools: CLO3D, Style3D) | Show the garment as-is; if demand emerges, it signals a different product line |
| Social features (comments, follows, public feed of garments) | "Sketchfab has a community" | Community without community is an empty room; garments of private shoppers have weak public-good dynamics | Share links with OG preview images; add community only if sharing data proves engagement |
| Email notifications for garment-ready | "Users might miss in-app notifications" | PROJECT.md explicitly decided in-app only; email noise hurts retention and adds deliverability complexity | In-app notification + badge; email stays auth-only |

## Feature Dependencies

```
[Auth/accounts]
    └──requires──> (nothing — foundation)

[Guided photo upload] ──requires──> [Auth]
[URL import] ──requires──> [Guided photo upload]   (upload is the fallback when scraping fails)

[3D generation job] ──requires──> [Photo upload or URL import]
[Job status UI + notification] ──requires──> [3D generation job]
[Quota enforcement] ──requires──> [Auth]
[Retry on failure] ──requires──> [Job status UI] + [Quota enforcement]

[Garment library] ──requires──> [3D generation job]
[Share links] ──requires──> [Garment library]
[Turntable video export] ──enhances──> [Share links]

[360° viewer] ──requires──> [3D generation job]
[Body presets] ──requires──> [360° viewer]
[Body sliders] ──requires──> [Body presets]
[Size selector] ──requires──> [LLM size extraction] + [Body presets]
[True-to-size scaling] ──requires──> [LLM size extraction] + [Body presets]
[Size comparison view] ──requires──> [True-to-size scaling]
[Measurement overlay] ──enhances──> [LLM size extraction]

[AR Quick Look export] ──requires──> [360° viewer] (glTF asset pipeline)
[Mobile-responsive viewer] ──enhances──> everything viewer-related
```

### Dependency Notes

- **Everything user-facing requires Auth:** Quotas, libraries, and share-link ownership all hang off accounts; PROJECT.md correctly mandates accounts from day one.
- **URL import requires upload as sibling, not successor:** Scraping is best-effort by nature (store sites change, anti-bot measures); the product must never have URL import as the only ingestion path.
- **Size selector requires LLM extraction BEFORE body presets can matter:** Scaling a garment to "true size M" is meaningless until the mannequin has a defined body; body presets and extraction land in the same or adjacent phases.
- **Body sliders conflict with preset-only pseudo-fit assumptions:** Sliders demand continuous blend-shape interpolation; build preset rig as interpolable from the start to avoid rework.
- **Share links enhance growth but conflict with nothing:** Lowest-risk v1 feature; ship early.

## MVP Definition

### Launch With (v1)

Minimum viable product — what's needed to validate the concept.

- [ ] Email/password auth + verification — foundation for library, quota, sharing (PROJECT.md)
- [ ] Guided front+back photo upload — core ingestion; validates the Option C thesis
- [ ] Paste-URL import (best-effort, major stores + OG fallback) — primary differentiator for shopper workflow; ship in v1 but behind upload reliability
- [ ] Async 3D generation with status UI + in-app notification — minutes-long jobs are the product's rhythm
- [ ] 360° orbit/zoom viewer with HDRI lighting, background control, load progress — the "wow"
- [ ] 4–6 body presets (slim → plus) — representation is table stakes (Zeekit precedent)
- [ ] LLM size extraction (category, region, sizes, measurements) + chart fallback — the "any store" universality
- [ ] Size selector S–XXL with visible update — core fit question
- [ ] Garment library with delete/rename — persistence justifies the account wall
- [ ] Visible free-tier quota — rails for v2 payments; fairness signal
- [ ] Public read-only share links — organic growth loop
- [ ] Retry-on-failure without quota charge — trust with a paid API under the hood

### Add After Validation (v1.x)

Features to add once core is working.

- [ ] Body sliders (height/chest/waist/hips) — trigger: users report presets "close but not me"
- [ ] Size comparison view — trigger: usage data shows repeated regeneration at different sizes
- [ ] Measurement overlay panel — trigger: users distrust fit; cheap trust-builder
- [ ] Turntable video export — trigger: share links get meaningful traffic; amplify the loop
- [ ] Improved URL scraping breadth — trigger: failed-scrape telemetry identifies high-traffic stores

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] Stripe payments on quota rails — v2 per PROJECT.md; rails built in v1
- [ ] AR Quick Look / USDZ export — high delight, real conversion pipeline effort; post-PMF
- [ ] 2D photorealistic try-on bolt-on — distinct tech; only if 360° value is validated
- [ ] Multi-garment outfits — large scope; engagement play for a mature product
- [ ] Self-hosted generation — revisit after API cost data exists

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Guided photo upload | HIGH | LOW | P1 |
| Auth + verification | HIGH | LOW | P1 |
| Async generation + status + notification | HIGH | MEDIUM | P1 |
| 360° orbit/zoom viewer + lighting + backgrounds | HIGH | MEDIUM | P1 |
| Body presets | HIGH | MEDIUM | P1 |
| LLM size extraction + fallback | HIGH | MEDIUM | P1 |
| Size selector | HIGH | LOW | P1 |
| Garment library | HIGH | LOW | P1 |
| Visible quota | MEDIUM | LOW | P1 |
| Share links | MEDIUM | LOW | P1 |
| URL import | HIGH | MEDIUM | P1 (best-effort) |
| Retry on failure | MEDIUM | LOW | P1 |
| Body sliders | HIGH | MEDIUM | P2 |
| Size comparison view | HIGH | MEDIUM | P2 |
| Measurement overlay | MEDIUM | LOW | P2 |
| Turntable video export | MEDIUM | MEDIUM | P2 |
| AR Quick Look export | MEDIUM | MEDIUM | P3 |
| Mobile viewer polish | HIGH | MEDIUM | P2 |
| Outfit composition | MEDIUM | HIGH | P3 |

## Competitor Feature Analysis

| Feature | Zyler | 3DLOOK YourFit | Zeekit / Walmart | Sketchfab | Meshy / Tripo | Our Approach |
|---------|-------|----------------|------------------|-----------|---------------|--------------|
| Input | Selfie/photo | Smartphone body scan | Own photo or ~50 preset models | Model upload | Text/image to 3D | Guided front+back photos + URL import (unique) |
| 3D garment from product photos | No (2D compositing) | Partial (brand-side 3D assets) | No (2D image mapping, ~80K graphic components) | Hosts only | Yes — generic objects | Yes — garment-specific pipeline (core bet) |
| Body representation | User photo | Scanned body measurements | ~50 diverse preset models | n/a | n/a | 4–6 blend-shape presets + later sliders |
| Size/fit intelligence | Visualization-focused | Strong (returns-reduction pitch) | Body-type matching | n/a | n/a | LLM extraction + regional normalization — works on any store |
| Viewer | n/a (flat images) | Photoreal stills | Flat interactive image | Orbit, annotations, lighting, embed, JS API | In-app viewer | Orbit/zoom, HDRI, backgrounds (Sketchfab-standard interactions) |
| Sharing | Retailer-embedded | Retailer-embedded | Walmart.com-native | Public links, no-account viewing, embed iframes, AR Quick Look | Export gating on paid tiers | Public no-auth share links in v1; AR/video later |
| Monetization | B2B SaaS (M&S, John Lewis) | B2B enterprise | Internal to Walmart | Freemium + subscriptions | Freemium credits (Tripo: 300/mo free; low-res/format gating on free) | Visible free quota in v1; Stripe in v2 |
| URL import | No | No | No | No | No | Yes — unclaimed differentiator (pattern proven by dropshipping importers) |

**Key landscape takeaways:**

1. **The 2D try-on space is crowded (Zyler, Zeekit/Walmart, Style3D, Perfect Corp) but the "2 photos → real 3D garment on adjustable body" niche is empty at consumer level.** 3D garment work (CLO3D, Browzwear, Style3D) is professional design software with pattern-based pipelines — not accessible to shoppers. Our positioning is genuinely differentiated; the risk is 3D quality, not competition.
2. **Every consumer AI tool uses visible credits/quotas.** Tripo's 300 free credits/month and feature gating (resolution, export formats) are the established pattern; hiding limits is anti-pattern.
3. **Sketchfab defines the viewer UX floor:** frictionless orbit, no-account viewing of shared content, embed code. Our share links should meet this bar.
4. **Zeekit's preset-model diversity (~50 models across heights, shapes, skin tones) set user expectations** for body representation; 4–6 presets is the practical indie minimum, sliders close the gap.

## MVP Recommendation

Prioritize (matches PROJECT.md build order):
1. Auth → upload → generation job → viewer (the skeleton loop)
2. LLM size extraction → size selector → body presets + binding (the fit loop)
3. Library → quota → share links (the retention/growth loop)
4. URL import in parallel with upload hardening (the differentiator)

Defer: cloth physics (research-grade), 2D try-on bolt-on (different stack), body scanning (competitor moat), payments (v2 rails already planned), AR export (post-PMF).

## Sources

- [TryThisFit — try-on app roundup](https://trythisfit.com) — consumer try-on app landscape (LOW confidence, marketing content)
- [3DLOOK YourFit overview](https://www.apparelviews.com) — body-scan + fit positioning, Gartner citation (MEDIUM)
- [M&S pilots Zyler](https://www.einpresswire.com) — Zyler deployment model (MEDIUM)
- [Walmart "Be Your Own Model" / Zeekit](https://www.facebook.com) — preset models, photo mapping approach (MEDIUM)
- [Virtual fitting tools market report](https://www.datainsightsreports.com) — competitor categorization (LOW confidence, market-report SEO)
- [Style3D virtual try-on](https://www.style3d.ai) — professional/2D try-on positioning (HIGH — official site)
- [VNTANA format support (glTF, GLB, CLO3D, Browzwear)](https://www.vntana.com) — garment 3D format standards (MEDIUM)
- [Three.js product configurator lessons](https://dev.to) — OrbitControls, HDRI, background patterns (MEDIUM)
- [Sketchfab embedding + AR Quick Look/USDZ/Scene Viewer support](https://www.atomwolf.org), [Sketchfab Download API](https://apis.io) — viewer/sharing standards (MEDIUM-HIGH)
- [Tripo free tier: 300 monthly credits; AI 3D tool freemium patterns](https://tasarim.ai) — quota conventions (MEDIUM, corroborates Meshy/Tripo pricing pages)
- Note: "paste product URL → 3D garment" workflow found in NO competitor across all searches — negative claim backed by repeated null results (MEDIUM confidence; absence of evidence, flagged for ongoing validation)

---
*Feature research for: web-based 3D garment visualization / virtual fitting*
*Researched: 2026-09-29*
