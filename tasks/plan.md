# Implementation Plan: Cook-sama

> Rev 2 — updated for the expanded spec (baking, parallel/wait steps, caveats,
> step photos, AI photo import). Follows the agent-skills lifecycle: this plan
> is the `/plan` output; `/build` executes `tasks/todo.md` top to bottom.

## Overview

Build the recipe/baking tracker defined in `docs/spec.md` in five phases of
vertical slices. Every phase ends at a checkpoint with the app deployable. The
sister's cookie recipe (`docs/recipes/brown-butter-chocolate-chip-cookies.md`)
is the seed recipe and the acceptance fixture for the structural features.

## Architecture Decisions

- **Next.js App Router on Vercel; Prisma with SQLite dev / Postgres prod** —
  one codebase for owner UI and public share pages.
- **Slug-as-capability sharing** — unguessable slug is the share mechanism;
  `PRIVATE`/`UNLISTED` visibility gates rendering; drafts never render publicly.
- **Flat step list with `meanwhile` + `kind: WAIT`** rather than a dependency
  graph — expresses "mix dry while the butter browns" and "chill 3 hours" with
  one boolean and one enum, keeping the editor simple.
- **Per-step `caveat` field** — the tribal knowledge ("butter must be cool or
  the egg cooks") gets dedicated, prominently-rendered storage instead of
  being buried in step text.
- **One Zod recipe schema, three consumers** (editor form, server actions, AI
  structured output) — the AI physically cannot produce a draft the editor
  can't open.
- **AI import via Anthropic SDK** (`claude-opus-4-8`, vision blocks +
  `messages.parse` with `zodOutputFormat`) — drafts land as `isDraft: true`
  for owner review; source photos kept for provenance; API mocked in tests.
- **Structured ingredients (decimal qty + unit + name)** for real scaling;
  pure-function domain core in `src/lib`.

## Dependency Graph

```
Prisma schema (Recipe, Ingredient, Step[+meanwhile/wait/caveat], StepPhoto,
               SourceImage, Tag, CookLog)
    │
    ├── Zod recipe schema ───────────────┐
    │       │                            │
    │       ├── Server actions (CRUD)    ├── AI extraction (vision + parse)
    │       │       │                    │        │
    │       │       └── Owner editor ◄───┴── Import review (drafts)
    │       │
    │       └── Public recipe view ── cook mode (meanwhile panels, timers)
    │                                      │
    └── Auth ── route protection           └── scaling lib (pure)
```

## Task List

### Phase 1: Foundation
- [ ] Task 1: Scaffold Next.js + Tailwind + Vitest + Playwright + CI
- [ ] Task 2: Prisma schema + migration + seed (sister's cookies with meanwhile/wait/caveats, + 2 simpler recipes)
- [ ] Task 3: Owner auth + protected route group

### Checkpoint: Foundation
- [ ] CI green; sign-in works; seed renders in Prisma Studio with correct step structure

### Phase 2: Core recipe management (owner)
- [ ] Task 4: Dashboard recipe list
- [ ] Task 5: Create recipe (meta) — Zod schema + action + form
- [ ] Task 6: Ingredient editing (groups, order, qty/unit/name/note)
- [ ] Task 7: Step editing — order, kind (active/wait), meanwhile toggle, timer, caveat
- [ ] Task 8: Edit + delete + slug generation
- [ ] Task 9: Images — hero upload + per-step photos (Vercel Blob / local dev)

### Checkpoint: Core management
- [ ] Owner can fully enter the sister's cookie recipe by hand on a phone-width viewport, including meanwhile steps, waits, caveats, and a step photo

### Phase 3: Recreate & share
- [ ] Task 10: Public recipe page `/r/[slug]` (visibility + draft gating, noindex)
- [ ] Task 11: Serving scaler (pure lib + UI)
- [ ] Task 12: Cook mode — large type, checkboxes, wake-lock, **meanwhile panels, wait timers, caveat callouts, step photos**
- [ ] Task 13: Cook log
- [ ] Task 14: Share affordances — copy link, copy ingredients (scaled), print view

### Checkpoint: Recreate & share
- [ ] E2E: visitor opens unlisted cookie recipe, sees "Meanwhile" panel + chill timer, scales, prints; PRIVATE + drafts 404

### Phase 4: AI photo import
- [ ] Task 15: Extraction lib — Anthropic SDK call (vision + structured output on the shared Zod schema), prompt assembly, canned-response unit tests
- [ ] Task 16: Import flow — upload photos + caveat text → draft recipe + SourceImages → redirect to editor
- [ ] Task 17: Draft review UX — draft badge, flagged-question caveats surfaced, publish action flips `isDraft`

### Checkpoint: AI import
- [ ] With a mocked model response: screenshot + caveats → draft with parallel/wait structure → review → publish
- [ ] One manual smoke test against the real API with the actual cookie screenshot

### Phase 5: Find & polish
- [ ] Task 18: Tags (assign, filter chips — "baking"/"cooking" among them)
- [ ] Task 19: Search by title/ingredient
- [ ] Task 20: Performance & a11y pass (Lighthouse mobile ≥ 90, keyboard, axe)
- [ ] Task 21: Production deploy (Vercel + Postgres + Blob + ANTHROPIC_API_KEY) and smoke test

### Checkpoint: Complete
- [ ] All success criteria in `docs/spec.md` checked; sister's recipe imported, reviewed, shared, and baked from cook mode

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| AI extraction hallucinates quantities or invents steps | High | Structured outputs pin the shape; prompt demands "flag uncertainty as caveat, don't guess"; drafts always human-reviewed before publish |
| `meanwhile`/`WAIT` model too weak for some recipe | Med | It covers the target recipe class (1 parallel track); revisit multi-track only if a real recipe demands it |
| Editor complexity creep (meanwhile toggles, caveats, photos per step) | High | Progressive disclosure: plain text step by default; kind/meanwhile/timer/caveat/photo behind an expand control |
| Scaling produces ugly numbers | Med | Fraction-formatting pure function unit-tested first |
| PRIVATE/draft leak via image URLs | High | Visibility checked in routes; e2e asserts 404; blob paths unguessable |
| Anthropic API cost/failure | Low | Owner-only feature; graceful error, nothing persisted on failure; mocked in CI |

## Open Questions

Carried from `docs/spec.md` and `docs/recipes/brown-butter-chocolate-chip-cookies.md`
— none block Phase 1–2; the recipe-specific ones must be answered before the
seed data is final (Task 2 seeds best-guess values with the caveats embedded).
