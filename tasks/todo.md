# Todo: Cook-sama

Tasks sized S/M per planning-and-task-breakdown. Work top to bottom; each task
leaves the app working. See `tasks/plan.md` for phases and `docs/spec.md` for
the contract.

## Task 1: Scaffold project

**Description:** Next.js 15 + TypeScript (strict) + Tailwind app with test and
quality tooling wired, plus a CI workflow running lint/typecheck/test/build.

**Acceptance criteria:**
- [ ] `npm run dev` serves a placeholder home page
- [ ] `lint`, `typecheck`, `test`, `test:e2e`, `build` scripts all run
- [ ] GitHub Actions workflow runs the four checks on push

**Verification:** all scripts green locally and in CI.
**Dependencies:** None.
**Files:** scaffold + `.github/workflows/ci.yml` · **Scope:** M

## Task 2: Data layer

**Description:** Prisma schema for Recipe, Ingredient, Step, Tag, CookLog per
spec data model; first migration; seed script with 3 realistic recipes.

**Acceptance criteria:**
- [ ] `npx prisma migrate dev` creates the schema on SQLite
- [ ] `npm run db:seed` inserts 3 recipes with grouped ingredients and steps
- [ ] Prisma client singleton exported from `src/lib/db.ts`

**Verification:** `npx prisma studio` shows seeded data; unit test loads a recipe with relations.
**Dependencies:** Task 1.
**Files:** `prisma/schema.prisma`, `prisma/seed.ts`, `src/lib/db.ts` · **Scope:** M

## Task 3: Owner auth

**Description:** Auth.js credentials provider for a single owner account
(email + bcrypt hash from env). Route group `(owner)` requires a session.

**Acceptance criteria:**
- [ ] Owner signs in at `/signin` and reaches `/dashboard`
- [ ] Logged-out request to any `(owner)` route redirects to `/signin`
- [ ] Wrong password shows an error, no account enumeration

**Verification:** e2e covering both paths.
**Dependencies:** Task 1.
**Files:** `src/lib/auth.ts`, `src/app/signin/`, middleware · **Scope:** M

## Task 4: Dashboard recipe list

**Description:** `(owner)/dashboard` lists recipes (image thumb, title, tags,
last cooked), newest first.

**Acceptance criteria:**
- [ ] Seeded recipes render newest-first
- [ ] Empty state prompts "Add your first recipe"

**Verification:** e2e assertion on seeded titles order.
**Dependencies:** Tasks 2, 3.
**Files:** dashboard page + `RecipeCard` component · **Scope:** S

## Task 5: Create recipe (meta)

**Description:** Zod schema + server action + form for title, description,
servings, times, source URL, visibility. Redirects to the recipe editor.

**Acceptance criteria:**
- [ ] Valid submit persists and redirects; recipe appears on dashboard
- [ ] Server rejects invalid input (missing title, servings < 1) with field errors
- [ ] Action rejects unauthenticated callers

**Verification:** unit test on schema; e2e create flow.
**Dependencies:** Task 4.
**Files:** `src/lib/schemas/recipe.ts`, action, form component · **Scope:** M

## Task 6: Ingredient editing

**Description:** Editor section for ingredient rows (quantity, unit, name,
note) with optional group headers and ordering. Enter key adds next row.

**Acceptance criteria:**
- [ ] Add/remove/reorder rows persists `sortOrder` and `groupName`
- [ ] Quantity accepts decimals and empty ("to taste")
- [ ] Round-trip: saved ingredients re-render in the same order/groups

**Verification:** unit test on ingredients schema; e2e round-trip.
**Dependencies:** Task 5.
**Files:** `IngredientEditor` component, schema, action · **Scope:** M

## Task 7: Step editing

**Description:** Ordered step list editing with reorder and optional
`timerMinutes` per step.

**Acceptance criteria:**
- [ ] Add/remove/reorder steps persists `sortOrder`
- [ ] Timer minutes optional and persisted

**Verification:** e2e round-trip in editor.
**Dependencies:** Task 5.
**Files:** `StepEditor` component, schema, action · **Scope:** S

## Task 8: Edit, delete, slugs

**Description:** Full edit of existing recipes; delete with confirm; slug
generated from title + random suffix on create, stable across edits.

**Acceptance criteria:**
- [ ] Editing any field persists; slug does not change
- [ ] Delete removes recipe and children; dashboard updates
- [ ] Two recipes with the same title get distinct slugs

**Verification:** unit test on slug generator; e2e edit + delete.
**Dependencies:** Tasks 6, 7.
**Files:** `src/lib/slug.ts`, edit page, delete action · **Scope:** M

## Task 9: Recipe image

**Description:** Hero image upload — Vercel Blob in prod, local uploads dir in
dev — shown on dashboard card and recipe page.

**Acceptance criteria:**
- [ ] Upload from editor persists and renders (next/image)
- [ ] Replacing an image works; non-image files rejected

**Verification:** manual upload in dev; unit test on file validation.
**Dependencies:** Task 8.
**Files:** upload action, `src/lib/storage.ts`, editor field · **Scope:** M

## Task 10: Public recipe page

**Description:** `/r/[slug]` renders title, meta, grouped ingredients, steps,
image. UNLISTED renders for anyone with `noindex`; PRIVATE 404s for non-owner.

**Acceptance criteria:**
- [ ] Logged-out visitor sees an UNLISTED recipe fully rendered
- [ ] PRIVATE recipe → 404 for logged-out, renders for owner
- [ ] `<meta name="robots" content="noindex">` present on unlisted pages

**Verification:** e2e both visibility paths.
**Dependencies:** Task 8.
**Files:** `src/app/r/[slug]/page.tsx`, visibility guard in `src/lib` · **Scope:** M

## Task 11: Serving scaler

**Description:** Pure scaling + fraction-formatting functions in `src/lib`,
then a servings stepper on the recipe page that rescales all quantities.

**Acceptance criteria:**
- [ ] `scaleQuantity` and `formatQuantity` handle halving, doubling, thirds (1.5 → "1½")
- [ ] Stepper updates every numeric ingredient; "to taste" rows unchanged
- [ ] Base servings restore exactly

**Verification:** thorough unit tests on the lib; e2e scale check.
**Dependencies:** Task 10.
**Files:** `src/lib/scaling.ts`, `ServingScaler` component · **Scope:** M

## Task 12: Cook mode

**Description:** Full-screen step-by-step view from the recipe page: one step
at a time in large type, ingredient checklist, next/prev, screen wake-lock
where supported.

**Acceptance criteria:**
- [ ] Usable at 375px width; step text legible at arm's length
- [ ] Ingredient checkboxes persist while navigating steps
- [ ] Wake-lock requested when supported; mode works without it

**Verification:** e2e navigation through a seeded recipe; manual phone check.
**Dependencies:** Task 11.
**Files:** `CookMode` component + route · **Scope:** M

## Task 13: Cook log

**Description:** Owner logs a cook (date, 1–5 rating, notes); history renders
newest-first on the recipe page (owner view).

**Acceptance criteria:**
- [ ] Entry persists and renders with date, stars, notes
- [ ] Only the owner can create or see logs
- [ ] Dashboard card shows "last cooked" date

**Verification:** e2e add-log flow; visibility assertion for logged-out.
**Dependencies:** Task 10.
**Files:** `CookLogForm`, `CookLogList`, schema + action · **Scope:** M

## Task 14: Share affordances

**Description:** Copy-link button, copy-ingredients (plain text, scaled)
button, and a print stylesheet for the recipe page.

**Acceptance criteria:**
- [ ] Copy-link puts the `/r/[slug]` URL on the clipboard with confirmation
- [ ] Copy-ingredients respects current scaling
- [ ] Print preview: one page for a typical recipe, no nav/buttons

**Verification:** e2e clipboard checks; manual print preview.
**Dependencies:** Task 11.
**Files:** `ShareButtons` component, `print.css` · **Scope:** S

## Task 15: Tags

**Description:** Assign tags in the editor (create-on-type); dashboard filter
chips.

**Acceptance criteria:**
- [ ] Tags persist m:n; re-using a name reuses the tag
- [ ] Clicking a chip filters the dashboard; filters combine with search

**Verification:** e2e tag + filter flow.
**Dependencies:** Task 8.
**Files:** `TagInput`, dashboard filter, schema/action · **Scope:** M

## Task 16: Search

**Description:** Dashboard search box matching recipe title and ingredient
names, case-insensitive.

**Acceptance criteria:**
- [ ] "miso" matches a recipe with miso only in ingredients
- [ ] No results shows a friendly empty state

**Verification:** e2e search assertions on seed data.
**Dependencies:** Task 15.
**Files:** search param handling on dashboard, query in `src/lib` · **Scope:** S

## Task 17: Performance & accessibility pass

**Description:** Audit public recipe page and editor: Lighthouse mobile ≥ 90,
keyboard-only editor operation, labels/roles on cook mode controls.

**Acceptance criteria:**
- [ ] Lighthouse mobile perf ≥ 90 on `/r/[slug]`
- [ ] Editor fully operable by keyboard
- [ ] No serious axe violations on recipe page and cook mode

**Verification:** Lighthouse + axe reports checked into `docs/`.
**Dependencies:** Tasks 12, 14.
**Files:** as found · **Scope:** S

## Task 18: Production deploy

**Description:** Vercel project with Postgres and Blob, env secrets, migration
on deploy; smoke-test from a second device.

**Acceptance criteria:**
- [ ] Production URL serves the app; owner can sign in and create a recipe
- [ ] Unlisted link opens on a phone that has never seen the app
- [ ] `.env` values documented in `docs/deploy.md` (names only, no secrets)

**Verification:** manual production smoke test of the three core journeys.
**Dependencies:** All prior.
**Files:** Vercel config, `docs/deploy.md` · **Scope:** M
