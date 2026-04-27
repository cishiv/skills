# mvp-builder

Bundled skill system: idea → detailed spec → MVP spec → built app → Railway deployment.

The skill body and routing live in [`SKILL.md`](SKILL.md). This README is just for browsing the directory.

## What's in here

- [`detailed-specification/`](detailed-specification/) — turn a brain dump or interview Q&A into a project-mode detailed spec. The full vision before any scope-cutting.

## What's not here yet

- `mvp-specification/` — scope cutter. Adds ACCEPTANCE_CRITERIA.
- `build-mvp/` — loops on ACCEPTANCE_CRITERIA until they pass. Claude Code only.
- `railway-deployment-{kitchensinkts,twotier,statix}/` — first-time deploy.
- `extend-features/` — sibling entry point for feature work on existing repos.

These are scoped on demand, not up front, per `META_SCOPE` in the Obsidian vault.

## Templates this system targets

- [`kitchen-sink-ts`](https://github.com/cishiv/kitchen-sink-ts) — single SSR deployable.
- [`kitchen-sink-twotier`](https://github.com/cishiv/kitchen-sink-twotier) — separate Hono backend + React client.
- [`statix`](https://github.com/cishiv/statix) — static markdown site.

All three carry the Phase 0 conventions (`SPECIFICATIONS/`, `SPEC_TEMPLATE.md`, lazy-init, etc.) that the skills depend on.
