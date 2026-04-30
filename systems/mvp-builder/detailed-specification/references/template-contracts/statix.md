# statix — SPEC_TEMPLATE.md snapshot

**Snapshot date:** 2026-04-30
**Repo:** `github.com/cishiv/statix`
**File:** `SPECIFICATIONS/SPEC_TEMPLATE.md`

Use this snapshot only as a fallback when the live template can't be reached. Notify the user when this fallback is in use.

## Structure summary

- Frontmatter fields: `spec_type`, `mode`, `name`, `date_started`, `template`, `status`, `parent_spec` (same as the other two)
- Sections (in order): `ACCEPTANCE_CRITERIA`, `Problem statement`, `User flows`, `Content model changes`, `Architecture hints`, `Out of scope`
- **No `Integrations` section** — statix has no env-dependent SDKs.
- **`Content model changes` instead of `Data model changes`** — statix has no DB; the equivalent concept is content frontmatter and file conventions under `docs/`.

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

Acceptance criteria are sectioned-numbered, tagged, and written as EARS-flavored prose with first-class pre-conditions. The build agent picks the verification mechanism per criterion from this repo's menu (command exiting 0, built page DOM matching a documented assertion, or — for `[USER_VERIFIES]` — manual sign-off). See `CLAUDE.md` for the full contract.

### 1. <Section name (group by user flow or domain)>

#### 1.1 <Short criterion title> [BLOCKING]

**Pre-conditions:** <world-state required before this criterion applies>

WHEN `<event>`, the system shall `<response>`.

#### 1.2 <Short criterion title> [BLOCKING]

**Pre-conditions:** <…>

IF `<unwanted condition>`, THEN the system shall `<response>`.

### 2. <Section name>

#### 2.1 <Short criterion title> [NICE_TO_HAVE] [USER_VERIFIES]

**Pre-conditions:** <…>

<EARS-flavored prose; this criterion will be verified by the user, not by an automated check.>

## Problem statement

<Why this feature exists. What user need it serves. 2–3 paragraphs maximum.>

## User flows

<Step-by-step description of how a user interacts with this feature. Reference design assets in `SPECIFICATIONS/DESIGN/DESIGN.md` where applicable.>

## Content model changes

<Frontmatter fields added or modified. Build pipeline changes. New file conventions under `docs/`. If no content-model changes, write "None.">

## Architecture hints

<Suggested file paths, integration usage, suggested patterns. Hints, not prescriptions — the agent has discretion.>

## Out of scope

<Items explicitly deferred. For MVP specs, link to `MVP_{YYYYMMDD}_DEFERRED.md`.>
```
