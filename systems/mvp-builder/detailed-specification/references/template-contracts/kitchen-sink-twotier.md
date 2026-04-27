# kitchen-sink-twotier — SPEC_TEMPLATE.md snapshot

**Snapshot date:** 2026-04-26
**Repo:** `github.com/cishiv/kitchen-sink-twotier`
**File:** `SPECIFICATIONS/SPEC_TEMPLATE.md`

Use this snapshot only as a fallback when the live template can't be reached. Notify the user when this fallback is in use.

## Structure summary

- Frontmatter fields: `spec_type`, `mode`, `name`, `date_started`, `template`, `status`, `parent_spec`
- Sections (in order): `ACCEPTANCE_CRITERIA`, `Problem statement`, `User flows`, `Data model changes`, `Architecture hints`, `Integrations`, `Out of scope`
- Integrations tracked: `auth`, `polar`, `r2`, `openrouter`

Structurally identical to `kitchen-sink-ts`. The split is informational — the architecture hints section will differ in content (workspaces, separate server/client/shared) but the spec format itself does not.

For a project-mode detailed spec, **omit the `ACCEPTANCE_CRITERIA` section entirely** — it's the MVP skill's responsibility.

## Literal template content

```markdown
---
spec_type: "<detailed | mvp>"
mode: "<project | feature>"
name: "<project name in project mode; feature name in feature mode>"
date_started: "<YYYY-MM-DD>"
template: "<kitchen-sink-ts | kitchen-sink-twotier | statix>"
status: "NOT_YET_IMPLEMENTED"
parent_spec: ""
---

<!--
Frontmatter usage:

- spec_type: "detailed" or "mvp". Detailed specs are produced by /detailed-specification (project) or /extend-features (feature). MVP specs are produced by /mvp-specification.
- mode: "project" for the first spec of a new repo, "feature" for additions to an existing repo.
- name: the project name (project mode) or the feature name (feature mode). Used in the filename.
- parent_spec: relative path from this file to the upstream spec it consumes. Empty only for project-mode detailed specs (which have no upstream — only user input). All other spec types must have parent_spec populated.

See SPECIFICATIONS/HOW_TO_USE_SPECIFICATION.md for filename conventions and the full lifecycle.
-->

# <Feature name>

## ACCEPTANCE_CRITERIA

- [ ] [BLOCKING] <verifiable assertion>
- [ ] [NICE_TO_HAVE] <verifiable assertion>

Each criterion must reduce to one of:
- A command that exits 0 (e.g. `bun run typecheck`, `bun run test`).
- An HTTP request whose response matches a documented Zod schema.

Untagged criteria default to [BLOCKING]. Build and extend agents loop up to 3 attempts per criterion before escalating to the user.

## Problem statement

<Why this feature exists. What user need it serves. 2–3 paragraphs maximum.>

## User flows

<Step-by-step description of how a user interacts with this feature. Reference design assets in `SPECIFICATIONS/DESIGN/DESIGN.md` where applicable.>

## Data model changes

<Tables added or modified. Drizzle schema changes. If no DB changes, write "None.">

## Architecture hints

<Suggested file paths, integration usage, suggested patterns. Hints, not prescriptions — the agent has discretion.>

## Integrations

- **auth**: KEEP | REMOVE | NOT_PRESENT
- **polar**: KEEP | REMOVE | NOT_PRESENT
- **r2**: KEEP | REMOVE | NOT_PRESENT
- **openrouter**: KEEP | REMOVE | NOT_PRESENT

## Out of scope

<Items explicitly deferred. For MVP specs, link to `MVP_{YYYYMMDD}_DEFERRED.md`.>
```
