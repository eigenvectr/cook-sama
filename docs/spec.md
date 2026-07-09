# Spec: Cook-sama — Recipe & Baking Tracking Website

> Living document. Rev 2 (2026-07-09): expanded from "cooking" to cooking **and
> baking**; added parallel/meanwhile steps, passive waits, per-step caveats,
> step photos, and AI photo-import (OCR + caveat processing). Reference test
> case: `docs/recipes/brown-butter-chocolate-chip-cookies.md`.

## Objective

Build a website where the owner tracks recipes — cooking *and* baking — so
that they can **recreate them reliably** (including the caveats a plain recipe
can't express) and **share them with friends**.

**Who uses it:**

- **The owner** (single user): captures recipes — by typing them in, or by
  photographing a source (Instagram screenshot, handwritten card, packaging)
  and letting AI extract it; logs each bake/cook; refines recipes over time.
- **Friends** (anonymous visitors): open a shared link, read the recipe, scale
  it, print it. No account, no login.

**The motivating example** (see `docs/recipes/brown-butter-chocolate-chip-cookies.md`):
capturing how the owner's sister makes brown-butter chocolate chip cookies.
The source is an Instagram screenshot, but her actual method has structure a
flat step list can't hold:

- *Parallelism* — sift and mix the dry ingredients **while** the butter browns.
- *Passive waits* — the browned butter must **cool** (~15 min) before the egg
  stage; the dough **chills** 1–3 hours.
- *Technique caveats* — whip the egg until foamy *first*, unlike the source
  which creams butter and sugar.
- *Substitutions* — "espresso powder" is really a C7 Vietnamese instant
  coffee packet.

**User stories:**

1. As the owner, I can create a recipe with structured ingredients
   (quantity / unit / name / note, in groups), ordered steps, servings, times,
   tags, photos, a source URL, and free-form notes.
2. As the owner, I can photograph a recipe (screenshot, card, package label),
   add my spoken/typed caveats, and have **AI extract a complete structured
   draft** — ingredients, steps with parallel/wait structure, caveats — which
   I review and edit before publishing. Source photos stay attached for
   provenance.
3. As the owner, steps can be marked **"meanwhile"** (runs alongside the
   previous step) or **"wait"** (passive: cooling, chilling, proofing, with a
   timer), and any step can carry a highlighted **caveat**.
4. As the owner, I can attach **photos to individual steps** (what "foamy"
   looks like, what properly browned butter looks like) and a hero image.
5. As a cook (owner or friend), **cook mode** walks me step-by-step on a
   phone: large type, ingredient checkboxes, screen wake-lock, "Meanwhile…"
   callouts, wait timers, caveats surfaced prominently, step photos inline.
6. As a cook, I can **scale servings** and quantities update.
7. As the owner, after cooking I log a **cook log** entry (date, rating,
   "what I'd change") so the recipe improves each iteration.
8. As the owner, I share via an **unlisted link**; anyone with it can view,
   scale, and print. Search and tag filters help me find recipes fast.

**Success looks like:** the sister's cookie recipe goes in from two photos plus
a paragraph of caveats, and three months later either of us bakes an identical
batch from cook mode without consulting the Instagram post.

## Assumptions

1. **Single-owner editing**; friends are view-only. No multi-user accounts in v1.
2. **Web app, mobile-first** — the primary cooking/baking surface is a phone.
3. **Stack:** Next.js (App Router) + TypeScript + Tailwind; Prisma; SQLite dev
   / Postgres prod; Vercel hosting.
4. **Auth:** one owner account via Auth.js credentials from env.
5. **Sharing:** per-recipe `PRIVATE` / `UNLISTED` visibility; unlisted slug
   links are the sharing capability; `noindex` on unlisted pages.
6. **AI extraction** uses the Anthropic API (official TypeScript SDK,
   `@anthropic-ai/sdk`), model **`claude-opus-4-8`**, sending photos as vision
   image blocks and forcing a schema-valid result with structured outputs
   (`client.messages.parse` + `zodOutputFormat` over the same Zod recipe
   schema the editor uses). Requires an `ANTHROPIC_API_KEY` env var; import is
   an owner-only feature so per-request cost is negligible.
7. **Images** stored via Vercel Blob (local dir in dev): hero image, step
   photos, and original source photos.
8. "Baking support" means the *structural* features above (waits/timers,
   parallel steps, precise quantities, oven temps in steps) — not a separate
   recipe type. Cooking vs. baking is just a tag.

## Tech Stack

- **Framework:** Next.js 15 (App Router, RSC), TypeScript strict
- **Styling:** Tailwind CSS
- **Data:** Prisma — SQLite (dev), Postgres (prod)
- **Auth:** Auth.js (NextAuth v5), single credentials-based owner account
- **AI import:** `@anthropic-ai/sdk`, `claude-opus-4-8`, vision + structured outputs (Zod)
- **Images:** Vercel Blob
- **Validation:** Zod schemas shared between forms, server actions, and AI extraction
- **Testing:** Vitest (unit) + Playwright (e2e)
- **Hosting:** Vercel

## Commands

```
Dev:    npm run dev
Build:  npm run build
Test:   npm test                 # vitest run
E2E:    npm run test:e2e         # playwright test
Lint:   npm run lint
Types:  npm run typecheck
DB:     npx prisma migrate dev
Seed:   npm run db:seed          # seeds the sister's cookie recipe + 2 others
```

## Project Structure

```
docs/               → Spec, ADRs, captured recipe references
docs/recipes/       → Human-readable recipe captures (pre-app provenance)
tasks/              → plan.md and todo.md
prisma/             → schema.prisma, migrations, seed script
src/app/
  (owner)/          → Authenticated: dashboard, editor, import review
  r/[slug]/         → Public/unlisted recipe view + cook mode + print view
src/components/     → Shared UI (folder per component)
src/lib/            → Domain logic: scaling, slugs, auth, db, extraction
src/lib/schemas/    → Zod schemas (single source of truth — forms, actions, AI)
tests/              → Vitest unit tests
e2e/                → Playwright specs
```

## Data Model

```
Recipe
  id, slug (unique, unguessable suffix), title, description
  servings int, servingsUnit string?        # "cookies", "people"
  prepMinutes int?, cookMinutes int?
  sourceUrl string?, sourceNote string?     # "@emijujuu on Instagram"
  imageUrl string?                          # hero image
  visibility enum: PRIVATE | UNLISTED
  isDraft boolean                           # true for unreviewed AI imports
  tags Tag[] (m:n)                          # includes "baking", "cooking"
  ingredients Ingredient[], steps Step[], cookLogs CookLog[]
  sourceImages SourceImage[]                # the photos a recipe was imported from
  createdAt / updatedAt

Ingredient
  id, recipeId, sortOrder
  quantity decimal?                         # nullable: "to taste"
  unit string?, name string
  note string?                              # "finely chopped", "C7 instant coffee, not espresso powder"
  groupName string?                         # "For the dry mix"

Step
  id, recipeId, sortOrder
  text string
  kind enum: ACTIVE | WAIT                  # WAIT = passive (cool, chill, proof, rest)
  meanwhile boolean                         # runs alongside the previous non-meanwhile step
  timerMinutes int?                         # cook mode offers a timer (waits & timed actives)
  caveat string?                            # highlighted warning/tip ("butter must be cool or the egg cooks")
  photos StepPhoto[]

StepPhoto
  id, stepId, url, caption string?, sortOrder

SourceImage
  id, recipeId, url, sortOrder              # original screenshots/cards used for import

CookLog
  id, recipeId, cookedAt date, rating int? (1–5), notes string?

Tag
  id, name unique
```

**Key contract decisions:**

- **`meanwhile` is an ordering flag, not a graph.** Steps stay a flat ordered
  list; a `meanwhile: true` step executes concurrently with the most recent
  step above it that isn't `meanwhile`. This expresses "mix dry while butter
  browns" without a dependency-graph editor. Cook mode renders consecutive
  meanwhile steps as a "Meanwhile:" panel on their anchor step.
- **`WAIT` steps get first-class treatment in cook mode**: a big timer, and
  the "Meanwhile" panel of any parallel prep. Chilling/proofing/cooling are
  where baking recipes actually fail — they're steps, not footnotes.
- **One Zod recipe schema, three consumers:** the editor form, the server
  actions, and the AI extraction (`zodOutputFormat`). The model literally
  cannot return a draft the editor can't open.
- **AI imports land as `isDraft: true`** and are never publicly visible until
  the owner reviews and publishes. Source images stay attached.
- Slug-as-capability sharing and decimal-quantity scaling carry over from Rev 1.

## AI Photo Import (pipeline)

1. Owner uploads 1+ photos and optionally types caveats
   ("she sifts the flour and mixes dry while the butter browns…").
2. Server action calls Claude (`claude-opus-4-8`) with the images as vision
   blocks + the caveat text, using structured outputs against the Zod recipe
   schema (which includes `kind`, `meanwhile`, `timerMinutes`, `caveat` per
   step). Prompt instructs the model to: OCR faithfully, weave caveats into
   the right steps, mark parallel work as `meanwhile`, mark passive time as
   `WAIT` with a timer, keep uncertainties as explicit `caveat` questions
   rather than guessing.
3. Draft recipe (`isDraft: true`) is created with source images attached;
   owner is redirected to the editor to review, answer flagged questions, and
   publish.
4. Failures (unreadable photo, refusal) surface as a friendly error; nothing
   is persisted.

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

- Folder-per-component, colocated tests; server components by default.
- Domain logic (scaling, fractions, slugs, extraction prompt assembly) as pure
  functions in `src/lib`.
- No `any`; Zod-inferred types flow from `src/lib/schemas`.

## Testing Strategy

- **Unit (Vitest):** scaling and fraction formatting; slug generation; the Zod
  recipe schema (including meanwhile/wait/caveat shapes); extraction response
  handling (given a canned model response, the right draft is built);
  visibility rules.
- **E2E (Playwright):** (1) owner creates a recipe with meanwhile + wait steps
  and sees cook mode render the "Meanwhile" panel and timer; (2) anonymous
  visitor opens an unlisted link and scales servings; (3) PRIVATE recipe and
  draft recipes 404 for anonymous visitors; (4) import review flow with a
  mocked Anthropic response.
- The AI call itself is mocked in tests; one manual smoke test against the
  real API before shipping the import feature.

## Boundaries

- **Always:** run lint/typecheck/test before commits; validate mutations
  server-side with Zod; check owner session on every write and on the import
  endpoint; mock the Anthropic API in automated tests.
- **Ask first:** adding dependencies beyond the listed stack; schema changes
  after first migration ships; changing the sharing model; adding friend
  accounts; any feature that calls the AI on behalf of anonymous visitors.
- **Never:** commit secrets (`ANTHROPIC_API_KEY`, DB URLs); expose PRIVATE or
  draft recipes through any route; auto-publish an AI import without owner
  review; remove failing tests to get green.

## Success Criteria

- [ ] Owner can create/edit/delete recipes with grouped ingredients, ordered steps, meanwhile flags, wait steps with timers, per-step caveats, and step photos.
- [ ] The sister's cookie recipe (seed data) renders correctly: steps 2–3 appear as "Meanwhile" alongside browning, cooling and chilling show timers, caveats are visually prominent.
- [ ] Photo import: given the Instagram screenshot + caveat text, a structured draft is created with parallel/wait structure and the C7-coffee substitution captured; owner reviews and publishes.
- [ ] Cook mode is usable one-handed at 375px: legible steps, checkboxes, wake-lock, timers, meanwhile panels, step photos.
- [ ] Serving scaler rescales all numeric quantities (unit-tested for halving/doubling/thirds).
- [ ] Unlisted links render for logged-out visitors; PRIVATE and draft recipes 404 for them.
- [ ] Cook log persists and displays newest-first.
- [ ] Print view fits a typical recipe on one page; Lighthouse mobile ≥ 90 on the recipe page.

## Open Questions

1. Recipe-specific: the four questions in `docs/recipes/brown-butter-chocolate-chip-cookies.md` (C7 amount, baking soda placement, when sugars go in, batch yield).
2. Should import accept multiple recipes per photo batch, or one recipe per import? (Assumed: one.)
3. Voice caveats (record audio → transcribe) — v1 assumes typed caveats only.
