# Roadmap: Clothing 3D Fitting Viewer

## Overview

This roadmap takes the project from an empty repo to a publicly deployed MVP where a shopper can turn two garment photos (or a product URL) into a real 3D garment on an adjustable mannequin at true-to-size scale. The structure follows the dependency chain the research validated: the 3D viewer is proven first (it sets the asset/performance budget every later phase must honor), then accounts and photo intake, then the paid 3D-generation pipeline (the engine), then LLM size extraction and scale calibration, then body morphs and pseudo-fit (the core "wow"), and finally the retention/growth loop — library, share links, URL import, AR — and public launch. Critical path: **2 → 3 → 5**. Phase 3 requires Meshy/Tripo API keys; Phase 2 requires an email provider key; Phase 4 requires an LLM key — none exist yet and sign-ups are flagged in those phases.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: 3D Viewer Foundation** - Orbit/zoom garment viewer with loading, lighting, and mobile-safe performance budget
- [ ] **Phase 2: Accounts & Photo Intake** - Email/password accounts with verification, plus guided front+back photo upload with cutout confirmation
- [ ] **Phase 3: 3D Generation Pipeline** - Async photo-to-3D pipeline with job status, notifications, validation gate, and server-side quota *(requires Meshy/Tripo API keys)*
- [ ] **Phase 4: AI Sizing & True-to-Scale** - LLM size extraction from product descriptions with scale calibration so garments display true-to-size *(requires LLM API key)*
- [ ] **Phase 5: Body Types & Fit Interaction** - Adjustable mannequin presets, size selector, and side-by-side size comparison with pseudo-fit binding
- [ ] **Phase 6: Library, Sharing & Launch** - Garment library, public share links, URL import, AR view, and public deployment

## Phase Details

### Phase 1: 3D Viewer Foundation
**Goal**: Users can load and inspect a 3D garment in a browser with full orbit controls, before any backend exists — de-risking the r3f/React 19 stack and locking the mobile performance budget that every later phase must meet
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: VIEW-01, VIEW-02, VIEW-03, VIEW-05
**Success Criteria** (what must be TRUE):
  1. User can orbit a sample garment 360 degrees and zoom with both mouse and touch
  2. Viewer shows a visible progress indicator until the model first renders
  3. Garment renders under HDRI lighting and the user can adjust the background
  4. Viewer works smoothly on a real mid-range Android phone — responsive layout, touch orbit, pinch zoom, within the documented asset budget (tri count, GLB size)
**Plans**: TBD
**UI hint**: yes

### Phase 2: Accounts & Photo Intake
**Goal**: Users can create verified accounts and submit garment photos through a guided intake flow that guarantees a clean cutout before any generation credits are spent
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03, INTAKE-01
**Success Criteria** (what must be TRUE):
  1. User can create an account with email and password and verify it via a link received in Gmail
  2. User can log in, log out, and remain logged in across browser sessions
  3. User can complete the guided front + back photo upload, with example thumbnails shown and invalid inputs rejected with clear feedback
  4. User confirms the background-removed cutout before submission, and bad source photos are caught by a pre-flight quality gate
**Plans**: TBD
**UI hint**: yes

> **External dependency:** Requires an email delivery account (e.g., Resend) and a database (e.g., Neon Postgres) — sign-ups happen during this phase; free tiers carry development.

### Phase 3: 3D Generation Pipeline
**Goal**: Users can turn uploaded photos into a real 3D garment model via a background pipeline they can watch, with failures retryable and cost protected by server-side quota — the product's engine
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: GEN-01, GEN-02, GEN-03, GEN-04, QUOTA-01, QUOTA-02
**Success Criteria** (what must be TRUE):
  1. User submits photos and generation runs as a background job — the UI never blocks for minutes
  2. User sees staged progress (queued → generating → finalizing) and receives an in-app notification when the garment finishes
  3. User can retry a failed generation without consuming quota
  4. Free-tier quota is enforced server-side per account per month, and the user sees remaining quota next to the create button
  5. Generated meshes pass an automated validation gate and are post-processed to the Phase 1 asset budget
**Plans**: TBD
**UI hint**: yes

> **External dependency:** Requires Meshy and/or Tripo API accounts (3D generation, ~$0.20-0.40/garment) and an Inngest account (background jobs) — sign-ups are a Phase 3 kickoff checklist item. Verify signup credits and webhook parameters at account creation.

### Phase 4: AI Sizing & True-to-Scale
**Goal**: Garments display at true-to-size scale — the system extracts sizing (category, region, sizes, measurements) from the product description and calibrates the mesh against it
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: FIT-02, FIT-04
**Success Criteria** (what must be TRUE):
  1. Pasting a product description yields structured sizing: category, region, available sizes S-XXL, and measurements in centimeters (regional sizes normalized)
  2. Garment displays at true-to-size scale on the mannequin, matching the extracted measurements
  3. Descriptions without parseable sizing fall back to a standard size chart and still produce a calibrated garment
**Plans**: TBD

> **External dependency:** Requires an LLM provider API key (Gemini Flash-class via Vercel AI SDK) — sign-up happens during this phase; per-call cost is fractions of a cent.

### Phase 5: Body Types & Fit Interaction
**Goal**: Users can see the garment on a body like theirs — adjustable mannequin presets, size selection, and side-by-side comparison, with the garment following the body via pseudo-fit binding (the core "wow")
**Mode:** mvp
**Depends on**: Phase 4 (calibrated garments) and Phase 1 (asset budget)
**Requirements**: FIT-01, FIT-03, FIT-05
**Success Criteria** (what must be TRUE):
  1. User can choose from 4-6 body presets (slim through plus-size) and the garment visibly follows the chosen body
  2. User can select a garment size (S-XXL) and see the display update accordingly
  3. User can compare two sizes side-by-side and see the difference
  4. Fit approximation is honestly communicated (spandex-hug behavior on extreme body sizes)
**Plans**: TBD
**UI hint**: yes

### Phase 6: Library, Sharing & Launch
**Goal**: The retention and growth loop — saved garment library, public share links, paste-a-URL import, AR viewing — and the app deployed publicly for real people
**Mode:** mvp
**Depends on**: Phase 3 (garments + notifications), Phase 5 (fit viewer)
**Requirements**: LIB-01, LIB-02, INTAKE-02, VIEW-04, PLAT-01
**Success Criteria** (what must be TRUE):
  1. User has a garment library with thumbnails and can open any saved garment in the viewer
  2. User can share any garment via a public read-only 360-degree link that viewers can open without an account
  3. User can paste a store product URL and the app fetches candidate images + description, gracefully falling back to upload when fetching fails
  4. User can open their garment in AR on their phone (Quick Look / Scene Viewer)
  5. App is deployed publicly and a real person can complete the full flow: sign up, create a garment, view it on a body, share it
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6
(Critical path: 2 → 3 → 5; Phase 4 slots between 3 and 5 because fit needs calibrated sizes)

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. 3D Viewer Foundation | 0/TBD | Not started | - |
| 2. Accounts & Photo Intake | 0/TBD | Not started | - |
| 3. 3D Generation Pipeline | 0/TBD | Not started | - |
| 4. AI Sizing & True-to-Scale | 0/TBD | Not started | - |
| 5. Body Types & Fit Interaction | 0/TBD | Not started | - |
| 6. Library, Sharing & Launch | 0/TBD | Not started | - |
