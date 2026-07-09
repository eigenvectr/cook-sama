# Cook-sama

A personal recipe-tracking website: capture the recipes you cook, recreate them
reliably (scaled servings, step-by-step cook mode, notes from past attempts),
and share them with friends via a simple link — no account needed to view.

## Status

Architecture phase. The project follows a spec-driven workflow:

| Document | Purpose |
|---|---|
| [`docs/spec.md`](docs/spec.md) | What we're building, tech stack, data model, boundaries, success criteria |
| [`tasks/plan.md`](tasks/plan.md) | Implementation plan: architecture decisions, phases, risks |
| [`tasks/todo.md`](tasks/todo.md) | Checklist of implementable tasks with acceptance criteria |

## Development workflow (agent-skills plugin)

This project is developed with the [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills)
plugin for Claude Code, which provides `/spec`, `/plan`, `/build`, `/test`,
`/review`, and `/ship` commands. It is installed locally in the repo (git-ignored):

```bash
git clone https://github.com/addyosmani/agent-skills.git
claude --plugin-dir ./agent-skills
```

Typical loop once the spec exists: `/plan` → `/build` (one task at a time,
test-driven) → `/review` → `/ship`.
