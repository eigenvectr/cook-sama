# Todo: Cook-sama

> Rev 2 — synced to the expanded spec (baking, meanwhile/wait steps, caveats,
> step photos, AI import). Work top to bottom; each task leaves the app
> working. See `tasks/plan.md` for phases, `docs/spec.md` for the contract.

## Task 1: Scaffold project

**Description:** Next.js 15 + TypeScript strict + Tailwind with test/quality
tooling and a CI workflow (lint, typecheck, test, build).

**Acceptance criteria:**
- [ ] `npm run dev` serves a placeholder page
- [ ] `lint`, `typecheck`, `test`, `test:e2e`, `build` scripts run
- [ ] GitHub Actions runs the checks on push

**Verification:** all scripts green locally and in CI.
**Dependencies:** None. · **Scope:** M

## Task 2: Data layer + seed

**Description:** Prisma schema per spec (Recipe, Ingredient, Step with
`kind`/`meanwhile`/`timerMinutes`/`caveat`, StepPhoto, SourceImage, Tag,
CookLog); first migration; seed script whose centerpiece is the sister's
cookie recipe from `docs/recipes/brown-butter-chocolate-chip-cookies.md`
(meanwhile steps 2–3, WAIT steps with timers, caveats, C7 ingredient note),
plus 2 simpler recipes.

**Acceptance criteria:**
- [ ] `npx prisma migrate dev` creates the schema on SQLite
- [ ] Seed inserts the cookie recipe with correct step flags and caveats
- [ ] Unit test loads it with relations and asserts the meanwhile/wait structure

**Verification:** Prisma Studio inspection + relation-loading unit test.
**Dependencies:** Task 1. · **Scope:** M

## Task 3: Owner auth

**Description:** Auth.js credentials provider, single owner account from env;
`(owner)` route group requires a session.

**Acceptance criteria:**
- [ ] Sign in at `/signin` → `/dashboard`; logged-out access redirects
- [ ] Wrong password errors without account enumeration

**Verification:** e2e both paths.
**Dependencies:** Task 1. · **Scope:** M

## Task 4: Dashboard recipe list

**Description:** `(owner)/dashboard` lists recipes (thumb, title, tags, last
cooked, draft badge), newest first.

**Acceptance criteria:**
- [ ] Seeded recipes render newest-first; drafts visibly badged
- [ ] Empty state prompts "Add your first recipe"

**Verification:** e2e on seeded titles.
**Dependencies:** Tasks 2, 3. · **Scope:** S

## Task 5: Create recipe (meta)

**Description:** Zod schema + server action + form for title, description,
servings (+ unit, e.g. "cookies"), times, source URL/note, visibility.

**Acceptance criteria:**
- [ ] Valid submit persists and redirects to the editor
- [ ] Server rejects invalid input with field errors; unauthenticated rejected

**Verification:** schema unit test; e2e create flow.
**Dependencies:** Task 4. · **Scope:** M

## Task 6: Ingredient editing

**Description:** Ingredient rows (qty, unit, name, note) with group headers
and ordering; Enter adds next row; decimals and empty qty ("to taste") allowed.

**Acceptance criteria:**
- [ ] Add/remove/reorder persists `sortOrder` + `groupName`
- [ ] Round-trip preserves order, groups, and notes (e.g. the C7 substitution note)

**Verification:** schema unit test; e2e round-trip.
**Dependencies:** Task 5. · **Scope:** M

## Task 7: Step editing (with meanwhile / wait / caveat)

**Description:** Ordered step editing. Each step: text; expandable extras for
kind (active/wait), meanwhile toggle, timer minutes, caveat text. Plain text
stays the default so simple recipes stay simple.

**Acceptance criteria:**
- [ ] Add/remove/reorder persists `sortOrder`
- [ ] `kind`, `meanwhile`, `timerMinutes`, `caveat` persist and round-trip
- [ ] A `meanwhile` step visually indents/attaches under its anchor step in the editor
- [ ] Schema rejects `meanwhile: true` on the first step

**Verification:** schema unit tests; e2e recreating the cookie recipe's steps 1–4.
**Dependencies:** Task 5. · **Scope:** M

## Task 8: Edit, delete, slugs

**Description:** Full edit; delete with confirm; slug = title + random suffix,
stable across edits.

**Acceptance criteria:**
- [ ] Edits persist; slug unchanged; delete cascades; duplicate titles get distinct slugs

**Verification:** slug unit test; e2e edit + delete.
**Dependencies:** Tasks 6, 7. · **Scope:** M

## Task 9: Images — hero + step photos

**Description:** Hero image upload plus per-step photos with captions
(Vercel Blob prod / local dir dev), shown in editor, recipe page, cook mode.

**Acceptance criteria:**
- [ ] Hero upload/replace works; non-images rejected
- [ ] Step photos attach to a specific step with optional caption and ordering

**Verification:** manual dev upload; file-validation unit test.
**Dependencies:** Task 8. · **Scope:** M

## Task 10: Public recipe page

**Description:** `/r/[slug]` renders meta, grouped ingredients, structured
steps (meanwhile grouping, wait styling, caveat callouts, step photos), hero.
UNLISTED renders with `noindex`; PRIVATE and `isDraft` 404 for non-owner.

**Acceptance criteria:**
- [ ] Logged-out sees UNLISTED fully rendered incl. meanwhile/wait/caveat presentation
- [ ] PRIVATE and draft → 404 logged-out; render for owner
- [ ] `noindex` meta present on unlisted pages

**Verification:** e2e all visibility paths.
**Dependencies:** Task 8. · **Scope:** M

## Task 11: Serving scaler

**Description:** Pure scaling + fraction formatting in `src/lib`; servings
stepper on the recipe page.

**Acceptance criteria:**
- [ ] Halving/doubling/thirds unit-tested (1.5 → "1½"); "to taste" unchanged
- [ ] Stepper updates all numeric quantities; base restores exactly

**Verification:** thorough lib unit tests; e2e scale check.
**Dependencies:** Task 10. · **Scope:** M

## Task 12: Cook mode

**Description:** Full-screen step-by-step: large type, ingredient checklist,
wake-lock, prev/next — plus **"Meanwhile" panels** (parallel steps shown with
their anchor), **wait timers** (start/pause countdown for WAIT or timed
steps), **caveat callouts** (visually loud), and inline step photos.

**Acceptance criteria:**
- [ ] Usable at 375px; checkboxes persist across navigation
- [ ] Cookie recipe: browning step shows sift+dry-mix as "Meanwhile"; cooling and chilling show timers; egg-foam caveat is prominent
- [ ] Wake-lock requested when supported; mode works without it

**Verification:** e2e walk through seeded cookie recipe; manual phone check.
**Dependencies:** Tasks 9, 11. · **Scope:** L → if it exceeds 5 files, split timers into a follow-up commit within the task

## Task 13: Cook log

**Description:** Owner logs date, 1–5 rating, notes; newest-first on recipe
page; "last cooked" on dashboard.

**Acceptance criteria:**
- [ ] Entry persists and renders; owner-only; dashboard shows last cooked

**Verification:** e2e add-log; logged-out visibility assertion.
**Dependencies:** Task 10. · **Scope:** M

## Task 14: Share affordances

**Description:** Copy-link, copy-ingredients (plain text, respects scaling),
print stylesheet.

**Acceptance criteria:**
- [ ] Copy-link/copy-ingredients work with confirmation; print fits one page, no chrome

**Verification:** e2e clipboard checks; manual print preview.
**Dependencies:** Task 11. · **Scope:** S

## Task 15: AI extraction lib

**Description:** `src/lib/extraction.ts` — assemble the prompt (OCR
faithfully; weave owner caveats into the right steps; mark parallel work
`meanwhile`, passive time `WAIT` + timer; flag uncertainty as `caveat`
questions, never guess) and call `claude-opus-4-8` via `@anthropic-ai/sdk`
with image blocks + `messages.parse` on the shared Zod recipe schema.

**Acceptance criteria:**
- [ ] Given a canned model response, returns a valid draft recipe object
- [ ] Refusal/parse failure surfaces a typed error; nothing persisted
- [ ] No API call in automated tests (client injected/mocked)

**Verification:** unit tests with canned responses.
**Dependencies:** Task 2 (schema). · **Scope:** M

## Task 16: Import flow

**Description:** `(owner)/import` — upload 1+ photos + optional caveat text →
extraction → create draft recipe (`isDraft: true`) with SourceImages attached
→ redirect to editor.

**Acceptance criteria:**
- [ ] Happy path creates a draft with structured steps and attached source photos
- [ ] Extraction failure shows a friendly error, persists nothing
- [ ] Owner-only; rejects unauthenticated

**Verification:** e2e with mocked extraction.
**Dependencies:** Tasks 9, 15. · **Scope:** M

## Task 17: Draft review & publish

**Description:** Draft badge + banner in editor listing flagged caveat
questions; "Publish" flips `isDraft` after validation.

**Acceptance criteria:**
- [ ] Draft never publicly visible until published
- [ ] Publish validates via the shared schema and clears the draft state

**Verification:** e2e import → review → publish → unlisted link renders.
**Dependencies:** Task 16. · **Scope:** S

## Task 18: Tags

**Description:** Tag assignment (create-on-type) + dashboard filter chips;
seed uses "baking"/"cooking".

**Acceptance criteria:**
- [ ] m:n persistence, name reuse; chips filter and combine with search

**Verification:** e2e tag + filter.
**Dependencies:** Task 8. · **Scope:** M

## Task 19: Search

**Description:** Dashboard search across title and ingredient names,
case-insensitive.

**Acceptance criteria:**
- [ ] "espresso" matches the cookie recipe via its ingredient; friendly empty state

**Verification:** e2e on seed data.
**Dependencies:** Task 18. · **Scope:** S

## Task 20: Performance & accessibility pass

**Description:** Lighthouse mobile ≥ 90 on `/r/[slug]`; keyboard-only editor;
axe on recipe page and cook mode; timers/announcements accessible.

**Verification:** Lighthouse + axe reports in `docs/`.
**Dependencies:** Tasks 12, 14, 17. · **Scope:** S

## Task 21: Production deploy

**Description:** Vercel + Postgres + Blob + `ANTHROPIC_API_KEY`; migrate on
deploy; smoke test: import the real cookie screenshot, publish, open the
unlisted link on a second device.

**Acceptance criteria:**
- [ ] Production import smoke test passes end-to-end
- [ ] Env var names (no values) documented in `docs/deploy.md`

**Verification:** manual production run of the three core journeys + import.
**Dependencies:** All prior. · **Scope:** M
