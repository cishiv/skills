# mvp-builder

Bundled skill system: idea → detailed spec → MVP spec → built app → Railway deployment.

The skill body and routing live in [`SKILL.md`](SKILL.md). This README is just for browsing the directory.

## What's in here

- [`detailed-specification/`](detailed-specification/) — turn a brain dump or interview Q&A into a project-mode detailed spec. The full vision before any scope-cutting.
- [`mvp-specification/`](mvp-specification/) — cut a detailed/extend spec down to the smallest valuable thing to ship. Writes EARS-flavored prose acceptance criteria. Handles both project and feature modes.
- [`build-mvp/`](build-mvp/) — sequential per-criterion loop against the live repo. One commit per criterion, `git reset` rollback per attempt, persistent verification artifacts, end-of-build report. Claude Code only.

## What's not here yet
- `railway-deployment-{kitchensinkts,twotier,statix}/` — first-time deploy.
- `extend-features/` — sibling entry point for feature work on existing repos.

These are scoped on demand, not up front.

## Templates this system targets

- [`kitchen-sink-ts`](https://github.com/cishiv/kitchen-sink-ts) — single SSR deployable.
- [`kitchen-sink-twotier`](https://github.com/cishiv/kitchen-sink-twotier) — separate Hono backend + React client.
- [`statix`](https://github.com/cishiv/statix) — static markdown site.

All three carry the shared conventions (`SPECIFICATIONS/`, `SPEC_TEMPLATE.md`, lazy-init, etc.) that the skills depend on.
