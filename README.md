# skills

Personal Claude skills.

## Skills

Standalone skills that work in both Claude Code and Claude Desktop. Invoke via slash command (e.g. `/interview-me`) or let Claude trigger them automatically when their description matches the request.

- **[interview-me](skills/interview-me/)** — Interviews you with batched, themed questions written to Obsidian before producing complex deliverables (specs, system prompts, architecture docs). Surfaces your own thinking rather than letting the AI fill in the blanks.
- **[update-next](skills/update-next/)** — Logs forward-looking work for the current project to `AI/NEXT.md` in Obsidian. Each project gets its own H3 with a checklist; the skill adds, completes, or edits items based on what the session surfaced.
- **[personal-brand-coach](skills/personal-brand-coach/)** — Multi-session coach for developing pillars, audience, positioning, anti-positioning, and a light voice guide. Writes canonical artifacts to `AI/PersonalBrand/` in Obsidian.
- **[generate-tone-reference](skills/generate-tone-reference/)** — Extracts a structured tone reference from writing samples (Obsidian notes, web pages, local repos). Produces `Writing/ToneReferences/{name}.md` with a machine-parseable frontmatter that downstream skills consume.
- **[linkedin-post-ideas](skills/linkedin-post-ideas/)** — Generates a week of LinkedIn post ideas anchored to your brand pillars and recent activity (GitHub commits, Obsidian changes, prior posts). Each idea is a checkbox plus a copy-paste invocation for `/linkedin-post-synthesis`.
- **[linkedin-post-synthesis](skills/linkedin-post-synthesis/)** — Drafts LinkedIn posts in your voice. Combines `personal-brand-coach` artifacts with a `generate-tone-reference` voice guide; produces 3 variants per topic with rationale.

The personal-brand / writing skills compose: `personal-brand-coach` + `generate-tone-reference` feed `linkedin-post-ideas` and `linkedin-post-synthesis`.

## Systems

A **system** is a bundled skill with sub-skills nested as folders. The top-level `SKILL.md` routes to the right sub-skill rather than doing the work itself. See [`systems/README.md`](systems/README.md) for the pattern.

- **[mvp-builder](systems/mvp-builder/)** — Idea → deployed MVP using the user's three template repos (`kitchen-sink-ts`, `kitchen-sink-twotier`, `statix`). Pipeline: `detailed-specification` → `mvp-specification` (optional) → `build-from-spec` → `railway-deployment`, with `extend-features` as a sibling entry point for adding features to existing repos.

## Format

Skills are stored in markdown (`SKILL.md`), not `.skill` format.
