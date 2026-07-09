# Spec: Cook-sama — Personal Recipe Tracking Website

## Objective

Build a website where the owner tracks the recipes they cook so that they can
**recreate them reliably** and **share them with friends**.

**Who uses it:**

- **The owner** (single user): adds and edits recipes, logs each time they cook
  one, tweaks and improves recipes over time.
- **Friends** (anonymous visitors): open a shared link, read the recipe, scale
  it to their servings, print it. No account, no login.

**User stories:**

1. As the owner, I can create a recipe with structured ingredients
   (quantity / unit / name), ordered steps, servings, times, tags, photos, a
   source URL, and free-form notes.
2. As the owner, I can find a recipe fast (search by title/ingredient, filter
   by tag) — "what was that noodle thing I made in March?"
3. As a cook (owner or friend), I can open **cook mode**: large-type
   step-by-step view with ingredient checkboxes and the screen kept awake,
   usable one-handed on a phone in the kitchen.
4. As a cook, I can **scale servings** and every ingredient quantity updates.
5. As the owner, after cooking I can add a **cook log** entry (date, rating,
   "what I'd change next time") so the recipe improves each iteration.
6. As the owner, I can share a recipe via an **unlisted link**; anyone with the
   link can view and print it, but it doesn't appear in search engines or to
   people without the link.
7. As a friend, I can view a shared recipe with zero friction and copy the
   ingredient list in one tap.

**Success looks like:** the owner enters a recipe once, cooks it three months
later from cook mode without consulting the original source, and a friend
recreates it from a texted link.

## Assumptions

> Surfaced per spec-driven-development; correct any of these and the spec gets
> updated before implementation.

1. **Single-owner editing** — friends are view-only; no multi-user accounts,
   comments, or friend logins in v1.
2. **Web app, mobile-first** — the primary cooking surface is a phone in the
   kitchen; no native app.
3. **Stack:** Next.js (App Router) + TypeScript + Tailwind CSS; Prisma ORM;
   SQLite for local dev, Postgres (Neon / Vercel Postgres) in production;
   deployed on Vercel.
4. **Auth:** one owner account via Auth.js (credentials provider with a single
   configured email + password hash from env). No sign-up flow.
5. **Sharing model:** per-recipe visibility of `private` (owner only) or
   `unlisted` (anyone with the slug link; `noindex`). A public gallery is out
   of scope for v1.
6. **Images** stored via Vercel Blob (or local `public/uploads` in dev); one
   hero image per recipe in v1.
7. Recipe **import from URL** (schema.org parsing) is a stretch goal, not v1.

## Tech Stack

- **Framework:** Next.js 15 (App Router, React Server Components), TypeScript strict
- **Styling:** Tailwind CSS
- **Data:** Prisma ORM — SQLite (dev), Postgres (prod)
- **Auth:** Auth.js (NextAuth v5), single credentials-based owner account
- **Images:** Vercel Blob storage
- **Validation:** Zod schemas shared between forms and server actions
- **Testing:** Vitest (unit) + Playwright (e2e)
- **Hosting:** Vercel

## Commands

```
Dev:    npm run dev
Build:  npm run build
Test:   npm test                 # vitest run
E2E:    npm run test:e2e         # playwright test
Lint:   npm run lint
Types:  npm run typecheck        # tsc --noEmit
DB:     npx prisma migrate dev   # apply schema changes locally
Seed:   npm run db:seed
```

## Project Structure

```
docs/               → Spec and architecture decision records
tasks/              → plan.md and todo.md (living planning docs)
prisma/             → schema.prisma, migrations, seed script
src/app/            → Routes (App Router)
  (owner)/          → Authenticated owner routes: dashboard, recipe editor
  r/[slug]/         → Public/unlisted recipe view + cook mode + print view
  api/              → Route handlers only where server actions don't fit
src/components/     → Shared UI components (one folder per component)
src/lib/            → Domain logic: scaling math, slug generation, auth, db client
src/lib/schemas/    → Zod schemas (single source of truth for validation)
tests/              → Vitest unit tests (mirrors src/ layout)
e2e/                → Playwright specs
```

## Data Model

```
Recipe
  id          cuid PK
  slug        unique, URL-safe, unguessable suffix (e.g. miso-ramen-x7k2f9)
  title       string
  description string?
  servings    int            # baseline for scaling
  prepMinutes int?
  cookMinutes int?
  sourceUrl   string?
  imageUrl    string?
  visibility  enum: PRIVATE | UNLISTED
  tags        Tag[] (m:n)
  ingredients Ingredient[]
  steps       Step[]
  cookLogs    CookLog[]
  createdAt / updatedAt

Ingredient
  id, recipeId FK, sortOrder int
  quantity    decimal?       # nullable: "salt to taste"
  unit        string?        # free-text unit ("g", "cup", "clove")
  name        string
  note        string?        # "finely chopped"
  groupName   string?        # section header: "For the sauce"

Step
  id, recipeId FK, sortOrder int
  text        string
  timerMinutes int?          # optional per-step timer hint

CookLog
  id, recipeId FK
  cookedAt    date
  rating      int? (1–5)
  notes       string?        # "used 2x garlic — keep it"

Tag
  id, name unique
```

**Key contract decisions** (per api-and-interface-design):

- The **slug is the sharing capability**: unlisted access = knowing the slug.
  Slugs embed a random suffix so they're unguessable; changing visibility to
  PRIVATE revokes access without deleting the recipe.
- Ingredient `quantity` is a decimal, never a string — scaling is arithmetic,
  not parsing. Display formatting (½, ⅓) is a pure function in `src/lib`.
- All mutations go through **server actions validated by Zod schemas**; the
  same schema drives the form and the server, so client and server can't drift.

## Code Style

```tsx
// src/lib/scaling.ts — pure domain logic, no framework imports
export function scaleQuantity(
  quantity: Decimal,
  baseServings: number,
  targetServings: number,
): Decimal {
  return quantity.mul(targetServings).div(baseServings);
}
```

- Components: PascalCase folder-per-component, colocated tests.
- Domain logic lives in `src/lib` as pure functions; components stay thin.
- Server components by default; `"use client"` only where interaction demands it.
- No `any`; Zod-inferred types flow from `src/lib/schemas`.

## Testing Strategy

- **Unit (Vitest):** scaling math, quantity formatting (fractions), slug
  generation, Zod schemas, visibility rules. High coverage here — this is the
  logic that makes recipes "recreatable."
- **E2E (Playwright):** the three critical journeys — (1) owner creates a
  recipe and sees it rendered, (2) anonymous visitor opens an unlisted link
  and scales servings, (3) anonymous visitor is blocked from a PRIVATE recipe
  and from all edit routes.
- Tests run in CI on every push; merge blocked on red.

## Boundaries

- **Always:** run `lint`, `typecheck`, and `test` before committing; validate
  every mutation server-side with Zod; check owner session in every server
  action that writes.
- **Ask first:** adding dependencies beyond the listed stack; schema changes
  after the first migration ships; changing the sharing/visibility model;
  anything that adds accounts for friends.
- **Never:** commit secrets or `.env`; expose PRIVATE recipes through any
  route (including image URLs); serve edit endpoints without auth; remove
  failing tests to get green.

## Success Criteria

- [ ] Owner can create, edit, and delete a recipe with grouped ingredients and ordered steps.
- [ ] Search by title/ingredient and tag filtering return correct results.
- [ ] Cook mode is usable on a 375px-wide phone: legible steps, checkable ingredients, screen wake-lock.
- [ ] Changing servings rescales all numeric quantities correctly (unit tests cover halving, doubling, thirds).
- [ ] An unlisted recipe link renders for a logged-out visitor; a PRIVATE one returns 404 for them.
- [ ] Cook log entries persist and display newest-first on the recipe page.
- [ ] Print view fits a typical recipe on one page.
- [ ] Lighthouse mobile performance ≥ 90 on the recipe view page.

## Open Questions

1. Should friends be able to leave a comment/reaction on a shared recipe, or is
   view-only enough for v1? (Spec assumes view-only.)
2. Is import-from-URL (paste a link, get a pre-filled recipe) wanted enough to
   pull into v1, or fine as a fast-follow?
3. Do you want a public "all my shared recipes" gallery page, or strictly
   link-by-link sharing?
