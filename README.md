# Cook-sama

A recipe-tracking website for cooking **and baking**: capture recipes (by hand
or by photographing a source and letting AI extract it), recreate them
reliably — parallel "meanwhile" steps, wait timers, technique caveats, step
photos, serving scaling, cook mode — and share them with friends via a simple
link, no account needed to view.

The founding use case: capturing exactly how my sister makes her brown-butter
chocolate chip cookies, caveats and all. See
[`docs/recipes/brown-butter-chocolate-chip-cookies.md`](docs/recipes/brown-butter-chocolate-chip-cookies.md).

## Status

Architecture phase complete (spec + plan). Next up: `/build` Task 1.

| Document | Purpose |
|---|---|
| [`docs/spec.md`](docs/spec.md) | What we're building: features, data model, AI import pipeline, boundaries, success criteria |
| [`tasks/plan.md`](tasks/plan.md) | Architecture decisions, dependency graph, 5 phases with checkpoints, risks |
| [`tasks/todo.md`](tasks/todo.md) | 21 implementable tasks with acceptance criteria |
| [`docs/recipes/`](docs/recipes/) | Recipes captured before the app exists (provenance + seed data source) |

## Development workflow (agent-skills plugin)

This project is developed with the [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills)
plugin for Claude Code, installed locally in the repo (git-ignored):

```bash
git clone https://github.com/addyosmani/agent-skills.git
claude --plugin-dir ./agent-skills
```

Its 8 slash commands are used **chronologically** through the lifecycle:

| # | Command | Phase | Status |
|---|---------|-------|--------|
| 1 | `/spec` | Define what to build → `docs/spec.md` | ✅ done (rev 2) |
| 2 | `/plan` | Break into tasks → `tasks/plan.md`, `tasks/todo.md` | ✅ done (rev 2) |
| 3 | `/build` | Implement one task at a time, test-driven | ⬜ next |
| 4 | `/test` | Prove it works | ⬜ |
| 5 | `/review` | Five-axis review before merge | ⬜ |
| 6 | `/webperf` | Audit web performance (pairs with Task 20) | ⬜ |
| 7 | `/code-simplify` | Clarity over cleverness | ⬜ |
| 8 | `/ship` | Deploy to production (pairs with Task 21) | ⬜ |

Tip: `/build auto` runs the whole task list autonomously after one plan
approval, still test-driven and committed per task.
