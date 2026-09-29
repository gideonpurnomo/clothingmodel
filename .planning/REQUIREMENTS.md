# Requirements: Clothing 3D Fitting Viewer

**Defined:** 2026-09-28
**Core Value:** A shopper can see a real garment they're considering, in 3D on a body like theirs, and judge how it would look and fit — from just two photos and a description.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Authentication & Accounts

- [ ] **AUTH-01**: User can create an account with email and password
- [ ] **AUTH-02**: User receives a verification email (deliverable to Gmail) and verifies their account
- [ ] **AUTH-03**: User can log in, log out, and stay logged in across sessions

### Garment Intake

- [ ] **INTAKE-01**: User can upload guided front + back photos with example thumbnails and client-side validation
- [ ] **INTAKE-02**: User can paste a store product URL and the app fetches candidate images + description (best-effort, with upload fallback)

### 3D Generation

- [ ] **GEN-01**: System turns the two photos into a 3D garment model via a 3D-generation API, as a background job
- [ ] **GEN-02**: User sees job status with progress feedback while generation runs
- [ ] **GEN-03**: User receives an in-app notification when their garment finishes processing
- [ ] **GEN-04**: User can retry a failed generation without consuming quota

### Viewer

- [ ] **VIEW-01**: User can orbit the garment 360° and zoom with mouse and touch
- [ ] **VIEW-02**: Viewer shows load progress before the first render
- [ ] **VIEW-03**: Viewer presents the garment with HDRI lighting and adjustable background
- [ ] **VIEW-04**: User can view their garment in AR on their phone (Quick Look / Scene Viewer)
- [ ] **VIEW-05**: Viewer works well on phones — responsive layout, touch-tuned orbit and pinch zoom, mobile GPU-safe asset budget

### Fit & Sizing

- [ ] **FIT-01**: User can choose from 4–6 body presets (slim → plus-size)
- [ ] **FIT-02**: System extracts size category, region, available sizes and measurements from the product description via LLM, with standard size chart fallback
- [ ] **FIT-03**: User can select garment size (S–XXL) and see the display update
- [ ] **FIT-04**: Garment displays true-to-size on the mannequin based on extracted measurements
- [ ] **FIT-05**: User can compare two sizes side-by-side

### Library & Sharing

- [ ] **LIB-01**: User has a saved garment library with thumbnails
- [ ] **LIB-02**: User can share any garment via a public read-only 360° link (viewers need no account)

### Quota

- [ ] **QUOTA-01**: Free-tier quota is enforced per account per month
- [ ] **QUOTA-02**: User sees remaining quota next to the create button

### Platform

- [ ] **PLAT-01**: App is deployed publicly and usable by real people

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Fit Experience

- **FITX-01**: Body sliders (height/chest/waist/hips) — continuous blend-shape interpolation beyond presets
- **FITX-02**: Measurement overlay panel — extracted garment measurements shown alongside mannequin

### Library

- **LIBX-01**: User can delete garments
- **LIBX-02**: User can rename garments

### Viewer & Growth

- **VIEWX-01**: Turntable video export — auto-rotating shareable clip

### Monetization

- **PAY-01**: Stripe monthly subscription for unlimited quota on the v1 quota rails

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Real cloth physics simulation (drape) | Research-grade (Neural Tailor / Sewformer); v3 destination, not v1 |
| 2D photorealistic try-on (user photo wearing garment) | Different tech stack (diffusion); splits focus from the 360° core |
| Body measurement from user photos ("scan yourself") | Competitor moat (3DLOOK), accuracy risk breaks trust, privacy surface |
| Self-hosted 3D generation (TRELLIS/Hunyuan3D) | Second product entirely; revisit after v1 validates demand |
| Fabric/material editor (recolor, texture swap) | Breaks the "real garment" promise; different product line |
| Social features (comments, follows, public feed) | Community without community is an empty room; share links first |
| Email garment-ready notifications | Decided: in-app notifications only; email stays auth-only |
| Native mobile apps | Web first; mobile-responsive viewer covers phones in v1 |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| (to be filled by roadmap) | | |

**Coverage:**
- v1 requirements: 23 total
- Mapped to phases: 0
- Unmapped: 23 ⚠️ (pending roadmap creation)

---
*Requirements defined: 2026-09-28*
*Last updated: 2026-09-28 after initial definition*
