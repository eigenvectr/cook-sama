# Implementation Plan: Cook-sama

## Overview

Build the recipe-tracking site defined in `docs/spec.md` in four phases, each a
set of vertical slices (schema + server action + UI per feature). Every phase
ends at a checkpoint with the app in a working, deployable state.

## Architecture Decisions

- **Next.js App Router on Vercel** — one codebase for owner UI and public
  sharing pages; RSC keeps the read-heavy public pages fast and simple.
- **Prisma with SQLite locally / Postgres in prod** — zero-setup local dev,
  managed durability in production; the schema is identical.
- **Slug-as-capability sharing** — an unguessable slug is the entire sharing
  mechanism. No share tokens table, no friend accounts. Visibility enum
  (`PRIVATE`/`UNLISTED`) gates rendering; flipping to PRIVATE revokes a link.
- **Structured ingredients (decimal quantity + unit + name)** rather than
  free-text lines — this is what makes serving-scaling and "copy ingredients"
  possible. Free-text is allowed via nullable quantity for "to taste" items.
- **Server actions + shared Zod schemas** for all writes — one validation
  source of truth; no hand-rolled API layer to keep in sync.
- **Pure-function domain core in `src/lib`** (scaling, fraction formatting,
  slugs) — unit-testable without booting the framework.

## Dependency Graph

```
Prisma schema (Recipe, Ingredient, Step, Tag, CookLog)
    │
    ├── Zod schemas (src/lib/schemas)
    │       │
    │       ├── Server actions (create/update/delete recipe, cook log)
    │       │       │
    │       │       └── Owner UI (editor, dashboard)
    │       │
    │       └── Public recipe view ── cook mode ── scaling (pure lib)
    │
    └── Auth (owner session) ── route protection
```

## Task List

### Phase 1: Foundation (walking skeleton)
- [ ] Task 1: Scaffold Next.js app with Tailwind, Vitest, Playwright, lint/typecheck scripts, CI workflow
- [ ] Task 2: Prisma schema + first migration + seed script with 3 sample recipes
- [ ] Task 3: Owner auth (Auth.js credentials, single account from env) + protected `(owner)` route group

### Checkpoint: Foundation
- [ ] `npm run build`, `lint`, `typecheck`, `test` all green in CI
- [ ] Logged-out visit to `/dashboard` redirects to sign-in; owner can sign in

### Phase 2: Core recipe management (owner)
- [ ] Task 4: Recipe list dashboard — seeded recipes render, newest first
- [ ] Task 5: Create recipe (slice: Zod schema + server action + editor form for title/meta)
- [ ] Task 6: Ingredient editing — grouped, ordered, quantity/unit/name/note rows
- [ ] Task 7: Step editing — ordered steps with reorder + optional timer minutes
- [ ] Task 8: Edit + delete recipe; slug generation with random suffix
- [ ] Task 9: Image upload (Vercel Blob prod / local dir dev) with hero display

### Checkpoint: Core management
- [ ] Owner can round-trip a full real recipe (create → edit → view) on a phone-width viewport
- [ ] E2E: owner creates recipe and sees it rendered

### Phase 3: Recreate & share (the point of the app)
- [ ] Task 10: Public recipe page at `/r/[slug]` honoring visibility (UNLISTED renders + noindex; PRIVATE → 404 for non-owner)
- [ ] Task 11: Serving scaler — pure scaling/fraction lib (heavily unit-tested) + UI stepper
- [ ] Task 12: Cook mode — step-by-step large-type view, ingredient checkboxes, screen wake-lock
- [ ] Task 13: Cook log — add entry (date, rating, notes) after cooking; newest-first history on recipe page
- [ ] Task 14: Share affordances — copy-link button, copy-ingredients button, print stylesheet

### Checkpoint: Recreate & share
- [ ] E2E: logged-out visitor opens unlisted link, scales servings, prints
- [ ] E2E: PRIVATE recipe 404s for logged-out visitor
- [ ] Review with owner before polish

### Phase 4: Find & polish
- [ ] Task 15: Tags — assign on edit, filter chips on dashboard
- [ ] Task 16: Search by title and ingredient name
- [ ] Task 17: Performance & a11y pass — Lighthouse mobile ≥ 90, keyboard/screen-reader check on editor and cook mode
- [ ] Task 18: Deploy to Vercel with Postgres + Blob; smoke-test production

### Checkpoint: Complete
- [ ] All success criteria in `docs/spec.md` checked off
- [ ] Production URL shared and tested from a second device

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Ingredient scaling produces ugly numbers (1.3333 cups) | Med | Fraction-formatting pure function with unit tests up front (Task 11 lib before UI) |
| Structured ingredient entry feels tedious vs. pasting text | High | Keep rows keyboard-fast (enter → next row); revisit paste-to-parse as fast-follow if painful |
| PRIVATE recipe leaks via image URL or cache | High | Visibility check in the page route *and* image paths namespaced per recipe; e2e test asserts 404 |
| Wake-lock API unsupported on some browsers | Low | Progressive enhancement — cook mode works without it |
| Scope creep (comments, accounts, meal plans) | Med | Boundaries section in spec: "ask first" before adding accounts/features |

## Open Questions

Carried from `docs/spec.md`: friend comments (assumed no), import-from-URL
(assumed fast-follow), public gallery (assumed no). Answers may re-order
Phase 4.
