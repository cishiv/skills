# mvp-builder

Bundled skill system: idea → detailed spec → (optional) MVP cut → built app → Railway deployment.

The skill body and routing live in [`SKILL.md`](SKILL.md). This README is just for browsing the directory.

## What's in here

- [`detailed-specification/`](detailed-specification/) — project-mode entry point. Turn a brain dump or interview Q&A into a detailed spec carrying full-vision `ACCEPTANCE_CRITERIA`. Both surfaces (Claude Code, claude.ai).
- [`extend-features/`](extend-features/) — feature-mode entry point. Reads existing repo state, anchors the new spec with a "What's already in place" preamble, supports post-hoc MVP backfill when the repo has prior work without an `IMPLEMENTED/MVP_*` artifact. Claude Code only.
- [`mvp-specification/`](mvp-specification/) — **optional** scope-cutting step. Cuts a detailed/extend spec down to the smallest valuable thing to ship. Writes a tighter `ACCEPTANCE_CRITERIA` + atomic `MVP_*_DEFERRED.md` sibling. Both modes, both surfaces.
- [`build-from-spec/`](build-from-spec/) — sequential per-criterion implementation loop. Consumes any spec with `ACCEPTANCE_CRITERIA` (detailed or MVP, project or feature). One commit per criterion, `git reset` rollback per attempt, persistent verification artifacts, end-of-build report. Claude Code only.
- [`railway-deployment/`](railway-deployment/) — first-time Railway provisioning + first deploy. Single parameterized skill covering all three templates (per-template differences in bundled `references/template-variants/`). Project mode only; subsequent deploys are `git push`. Requires Railway MCP and `web_fetch`. Claude Code only.

## Pipelines

```
Project mode:  detailed-specification → [mvp-specification?] → build-from-spec → railway-deployment
Feature mode:  extend-features        → [mvp-specification?] → build-from-spec → git push
```

`/mvp-specification` is optional. Skip it when the detailed/extend spec is already buildable as-is; invoke it when scope-cutting matters.

## Templates this system targets

- [`kitchen-sink-ts`](https://github.com/cishiv/kitchen-sink-ts) — single SSR deployable.
- [`kitchen-sink-twotier`](https://github.com/cishiv/kitchen-sink-twotier) — separate Hono backend + React client.
- [`statix`](https://github.com/cishiv/statix) — static markdown site.

All three carry the shared conventions (`SPECIFICATIONS/`, `SPEC_TEMPLATE.md`, lazy-init, etc.) that the skills depend on.
