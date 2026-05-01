---
name: extend-features
description: Use this skill when the user wants to add a new feature to an existing repo built from one of the three templates (kitchen-sink-ts, kitchen-sink-twotier, statix). Trigger on phrases like "extend this project", "add a feature", "I want to add X", or any time the user describes a new capability for a repo that already has implemented work. This skill produces a feature-mode detailed spec (`DETAILED_{FEATURE_NAME}_{YYYYMMDD}.md`) with `ACCEPTANCE_CRITERIA` covering the feature's full vision. The output flows to either `/mvp-specification` (when scope-cutting is wanted) or directly to `/build-from-spec`. Sibling to `/detailed-specification` — that one handles project mode (new repos), this one handles feature mode (existing repos). Claude Code only — needs to read existing repo state. Refuses gracefully and asks when invoked against a brand-new repo or a non-template repo.
---

# extend-features

This skill takes a description of a new feature and produces a **feature-mode detailed specification** anchored to the existing repo's state. The output sits alongside the prior implemented specs, references what's already in place via a "What's already in place" preamble, and carries `ACCEPTANCE_CRITERIA` covering the feature's full vision.

`/extend-features` is the feature-mode entry point — sibling to `/detailed-specification` (project mode). The two skills produce structurally identical specs; the differences are: this one **reads existing repo state** to anchor its output, can **backfill** a missing MVP spec when the repo has prior work without an `IMPLEMENTED/MVP_*` artifact, and writes a **preamble** describing what the feature builds on.

The output is consumable by `/mvp-specification` (when scope-cutting is wanted) or `/build-from-spec` directly (when the feature is small enough to ship as-is). `/mvp-specification` is optional.

## Boundaries

- **Claude Code only.** Needs to read existing repo state — file system, git history, prior `IMPLEMENTED/*.md` specs. claude.ai chat doesn't have the access this needs.
- **Feature mode only.** For new projects, use `/detailed-specification` instead. If invoked against a brand-new repo (no implemented work, no prior specs, just template scaffolding), the skill asks the user whether they meant project mode and offers to defer to `/detailed-specification`.
- **Bulk extends allowed.** Multiple in-flight `DETAILED_{FEATURE_NAME}_*.md` files in `NOT_YET_IMPLEMENTED/` are fine. Each invocation produces one new feature spec; the user can run the skill multiple times before building any of them.
- **`ACCEPTANCE_CRITERIA` is required.** Same contract as `/detailed-specification`'s post-amendment behavior — sectioned-numbered, EARS-flavored prose, with pre-conditions. Required because `/build-from-spec` may consume this spec directly.
- **Relaxed pre-flight.** This skill only writes to `SPECIFICATIONS/NOT_YET_IMPLEMENTED/`. Working tree state doesn't matter for correctness; uncommitted changes are flagged as a risk (the skill reads current code state to anchor the spec, and dirty changes can confuse that anchoring) but not refused.
- **Lower-friction gap analysis than `/detailed-specification`.** The repo already exists and has been thought through; the skill should ask fewer questions than the project-mode equivalent and be more aggressive about inferring from existing state.
- **No silent decisions.** Every fact in the spec traces to user input, a follow-up answer, an explicit `[OPEN: ...]` marker, or a claim derived from existing repo state (which is cited).
- **Generic-repo aware.** The skill works best on repos built from the three templates, but if invoked against a non-template repo (or a fork), it asks the user how to proceed rather than refusing outright.

## Workflow

### Step 1 — Verify surface and locate the target repo

Confirm the skill is running in Claude Code with filesystem access. If not, refuse — direct the user to invoke from Claude Code in the target repo's working directory.

Find the repo root by walking up from the cwd to find `SPECIFICATIONS/`. If walking up doesn't find it within 3 levels, ask the user where the target repo is (path).

### Step 2 — Detect template

Read the repo's `CLAUDE.md`. The first heading typically declares the template (`# kitchen-sink-ts — Ways of Working`).

- **Match against one of `kitchen-sink-ts`, `kitchen-sink-twotier`, `statix`** → proceed silently.
- **No match (Rails repo, plain Bun, fork of someone else's work, etc.)** → ask the user. Offer two paths: (a) proceed treating the repo as generic, with the caveat that template-aware affordances won't work as cleanly; (b) refuse and direct the user to a supported template. Honor their pick.

### Step 3 — Gather inputs

**Always ask the user explicitly where their input lives.** Acceptable input shapes:

- Feature description in chat ("I want to add X — it should do Y").
- Interview Q&A files from `CLAUDE_Q&A/ANSWERED/`.
- Raw markdown brain dumps.
- Voice notes (transcribed).
- Existing notes referenced by path (Obsidian or filesystem).
- Pasted text directly in chat.
- PDFs and images (mockups, competitor screenshots).
- Any combination.

Suggesting candidate Q&A files is allowed when Obsidian MCP is available, `CLAUDE_Q&A/ANSWERED/` exists, and `/interview-me` is installed. Otherwise just ask.

#### Refusal: input has no buildable feature

Same bar as `/detailed-specification` — the input must contain at minimum: one user flow (or extension to an existing flow) and a stated reason for the feature. Two-paragraph refusal if below: state what's missing concretely; suggest `/interview-me` and 2-3 specific aspects to develop.

### Step 4 — Confirm feature name

Read the input. If the user named the feature, use it. Otherwise propose an uppercase snake_case name based on the input (e.g. `MONETIZATION`, `TEAM_ACCOUNTS`, `DARK_MODE`). Tell the user: "I'll call this `<NAME>` — change?" Proceed on confirmation.

`FEATURE_NAME` is uppercase snake_case.

### Step 5 — Acquire the spec template

Same chain as `/detailed-specification`:

1. **Filesystem.** Read the repo's `SPECIFICATIONS/SPEC_TEMPLATE.md` directly.
2. **GitHub MCP / `web_fetch`** — fallbacks if the repo isn't fully on disk.
3. **Bundled snapshot** — under this skill's `references/template-contracts/<template>.md` if shared with `/detailed-specification`, otherwise the live template should always be reachable since the skill is Claude Code only.

If a bundled snapshot is used, tell the user the snapshot date and recommend verifying against the live template before committing.

### Step 6 — Discover existing repo state (relevance-matched)

Load **once-per-invocation**:

- Full `CLAUDE.md` — Principles + agent-writable Reference section.
- `DATABASEMODEL.md` if it exists.

For `IMPLEMENTED/*.md` specs, **don't load all of them**. Match by relevance to the proposed feature:

- **Token overlap.** The feature's name, key nouns, and described flows. If the feature mentions "billing" and an `IMPLEMENTED/IMPLEMENTED_MVP_PAYMENTS_*.md` exists, that's relevant.
- **File-map overlap.** If the feature touches `entries/` and a prior spec implemented `entries/`, that's relevant.
- **Section overlap.** Cross-reference the feature against prior specs' "User flows" and "Out of scope" sections.

Load only the matched specs. Cap at the most recent N (e.g. 3) per matched topic. The user can override by naming a specific prior spec.

For `NOT_YET_IMPLEMENTED/*.md` specs (work-in-progress), check titles and summaries. Load fully if any titles overlap with the proposed feature — to surface contradictions before committing.

### Step 7 — Backfill detection

Detect whether the repo has "materially diverged from the base template" without a prior `IMPLEMENTED/MVP_*` spec recording it.

Combination of signals — strongest signal wins:

- **Git history.** Number of commits since the initial template commit. If the repo has ≥10 substantive commits beyond the template scaffold, prior work probably happened.
- **File-map diff.** Files outside the template's expected scaffold paths (e.g. a `src/routes/_protected/dashboard/` not present in the template).
- **DB schema.** New tables in `src/lib/db/schema.ts` (or twotier-equivalent) beyond `users`, `sessions`, `accounts`, `verifications`, `subscriptions`, `uploads` (the scaffolded set).
- **`SPECIFICATIONS/IMPLEMENTED/` count.** If this is empty but the repo has clear prior work per the above signals, that's the backfill trigger.

If signals point to "materially diverged + no IMPLEMENTED MVP spec" → invoke the backfill workflow (Step 8). Otherwise skip to Step 9.

The user can always override the detection by saying "yes this repo has prior work, please backfill" or "no this repo is fresh, skip backfill."

### Step 8 — Backfill workflow (when needed)

When backfill is triggered, the skill produces a **post-hoc MVP specification** for the existing implemented work, so this feature spec has a `parent_spec` to point at and downstream skills have an artifact to read.

Workflow:

1. **Scan the repo** to reconstruct what was built. Sources:
   - `src/routes/` (or `server/` + `client/` for twotier; `src/components/` + `docs/` for statix) → user flows.
   - `src/lib/db/schema.ts` (or equivalent) → data model.
   - `.env.example` + lazy-init imports → integrations (which are KEEP / REMOVE).
   - Existing tests → ACCEPTANCE_CRITERIA hints (existing tests are the de facto criteria the original author thought were worth verifying).
   - `package.json` scripts → architecture hints.
2. **Draft a reconstructed `MVP_{YYYYMMDD}_SPEC.md`** with:
   - Frontmatter (`spec_type: mvp`, `mode: project`, `name: <PROJECT_NAME>` — derived from `package.json` name or repo name, `date_started: <today>`, `template`, `status: IMPLEMENTED`, `parent_spec: ""`).
   - Full sections (Problem statement, User flows, Data model changes, Architecture hints, Integrations, Out of scope) — all inferred from code, with citations.
   - `ACCEPTANCE_CRITERIA` — inferred from existing tests + observed behavior.
3. **Present the draft to the user** for confirmation/edits. Ask whether to:
   - Accept as-is.
   - Edit before writing.
   - Split into multiple specs (if the prior work clearly has multiple "phases" — e.g. auth shipped separately from billing). The skill is pragmatic: it asks the user how to slice rather than imposing a heuristic.
4. **Write directly to `SPECIFICATIONS/IMPLEMENTED/IMPLEMENTED_MVP_{YYYYMMDD}_SPEC.md`** (or one file per phase if split). Naming follows the standard convention; no special "backfilled" marker in the filename.
5. **Mention in the deploy report (if any)** that the spec was backfilled, so future readers know.

After backfill, proceed with Step 9 (gap analysis for the new feature) using the just-written backfill spec as part of the existing-repo-state context.

### Step 9 — Gap analysis (focused on the feature)

Compare the user's input against the spec template's required sections, but **lower friction than `/detailed-specification`**:

- Skip questions whose answer is obvious from existing repo state (don't ask "what auth library?" when `CLAUDE.md` says BetterAuth).
- Skip questions about template-provided affordances (the user already chose this template; assume they accept its conventions).
- Focus on what's actually new: the feature's own data model, user flows, integrations transitions, and AC.

Sections to cover:

- **What's already in place** (preamble — see Step 12) — derived from existing-repo-state context, not asked.
- **Problem statement** — why this feature now.
- **User flows** — at least one happy path end-to-end for the feature.
- **Data model changes** — new tables, new columns on existing tables, schema diffs.
- **Architecture hints** — file paths the feature will touch, integration usage, suggested patterns.
- **Integrations** — KEEP / REMOVE / NOT_PRESENT, with **explicit transition annotations** if the feature changes a prior decision (e.g. "polar: was REMOVE in original MVP; this feature requires re-enabling — see Step 10 cross-check").
- **`ACCEPTANCE_CRITERIA`** — sectioned-numbered, EARS-flavored prose, with pre-conditions. Cover the feature's full vision; `/mvp-specification` will cut if invoked.
- **Out of scope** — items explicitly deferred for this feature.

### Step 10 — Cross-spec contradiction check

Cross-reference the feature against:

- All loaded `IMPLEMENTED/*.md` specs' "Out of scope" sections.
- All loaded specs' Integrations sections (especially items marked `REMOVE`).
- All loaded specs' user flows and data model.

For each finding:

- **Out-of-scope from a prior spec** — surface to the user, but **don't refuse**. Out-of-scope-in-prior is not out-of-scope-in-perpetuity. The user explicitly acknowledges they're extending past a prior boundary.
- **Re-enabling a `REMOVE`'d integration** — surface for explicit acknowledgement. The new feature's Integrations section gets the transition annotation: "polar: was REMOVE in original MVP; this feature requires re-enabling."
- **Schema modification** (changing an existing table, not just adding new ones) — surface only when the user needs to mediate (e.g. type change on an existing column, drop a column). Adding a new column to an existing table proceeds silently.
- **Conflict with a `NOT_YET_IMPLEMENTED/*` spec** — surface and confirm with the user. They may need to ship the in-flight spec first, or revise this feature to coexist.

### Step 11 — Follow-up questions

Default mode is **single batch**. The user can opt into iterative.

Question delivery (inline vs Obsidian Q&A) follows the same per-invocation user-choice pattern as `/detailed-specification`. Default is inline (Claude Code, where chat is more interactive than for project-mode initial scoping).

**Hard limit: 20 questions per batch.** But aim much lower — feature-mode work usually needs <10 questions because the existing repo answers most of the obvious ones.

`[OPEN: <question>]` markers and weak-answer handling are identical to `/detailed-specification`.

### Step 12 — Pre-write validation

Validate:

- The target repo has `SPECIFICATIONS/NOT_YET_IMPLEMENTED/`.
- No same-day collision with `DETAILED_{FEATURE_NAME}_{YYYYMMDD}.md`.

#### Same-day collision policy: soft

Different from `/detailed-specification`'s hard refuse. If a same-day spec exists, surface it to the user and ask: overwrite, pick a different name, or proceed with the existing one. Allow collisions when the user explicitly confirms — they may genuinely mean a different feature even if the name collides.

### Step 13 — DEFERRED awareness

`MVP_*_DEFERRED.md` files in `NOT_YET_IMPLEMENTED/` list items the user previously deferred. The skill **does not auto-suggest** items from DEFERRED. The skill **does** mention them when the user references them, where reference means any of:

- The user explicitly names a `MVP_*_DEFERRED.md` file path.
- The user's feature description matches a deferred item exactly (token overlap with the deferred item's text).
- The user asks "what's been deferred" or "what was cut" or similar.

When mentioning, the skill cites the source DEFERRED file and the specific item, then asks whether the user wants to use that as the basis for this feature spec.

### Step 14 — Write the spec

**Filename:** `DETAILED_{FEATURE_NAME}_{YYYYMMDD}.md`

**Output location:** `SPECIFICATIONS/NOT_YET_IMPLEMENTED/<filename>` directly in the target repo.

**Frontmatter** (exactly this shape):

```yaml
---
spec_type: "detailed"
mode: "feature"
name: "<FEATURE_NAME>"
date_started: "<YYYY-MM-DD>"
template: "<chosen template>"
status: "NOT_YET_IMPLEMENTED"
parent_spec: "<relative path to upstream IMPLEMENTED detailed spec, falling back to MVP spec>"
---
```

`parent_spec` resolution:

1. Look in `SPECIFICATIONS/IMPLEMENTED/` for the most relevant `IMPLEMENTED_DETAILED_*.md` (project-mode detailed spec, or a prior feature-mode detailed spec the new feature directly builds on). Prefer the project-mode one if both exist.
2. If no implemented detailed spec exists (e.g. backfill case, or older repos that only got MVP specs), fall back to `IMPLEMENTED_MVP_*.md`.
3. The path is relative to the spec being written: typically `../IMPLEMENTED/<filename>`.

**Drop the frontmatter usage comment block** that the bundled template snapshot contains. The produced spec jumps straight from the closing `---` of the frontmatter to the `# <Feature name>` heading.

**Sections** (in this order):

1. **What's already in place** — preamble. Brief description of the existing implemented behavior the feature builds on. Cites prior `IMPLEMENTED/*` specs by relative path. This section is mandatory in feature mode and gives downstream skills (`/mvp-specification`, `/build-from-spec`) explicit handoff context so they don't rebuild from scratch.
2. **`ACCEPTANCE_CRITERIA`** — sectioned-numbered, EARS-flavored prose, with pre-conditions. Covers the feature's full vision.
3. **Problem statement** — why this feature, what user need it serves.
4. **User flows** — happy paths and key error paths for the feature.
5. **Data model changes** (or "Content model changes" for `statix`) — new tables, new columns, schema diffs from the existing model.
6. **Architecture hints** — file paths, integration usage, suggested patterns.
7. **Integrations** (omit for `statix`) — KEEP / REMOVE / NOT_PRESENT with explicit transition annotations where applicable.
8. **Out of scope** — what's explicitly deferred for this feature.

The skill must not invent decisions silently. Every fact in the spec traces to user input, a follow-up answer, an explicit `[OPEN: <question>]` marker, or a claim derived from existing repo state (which is cited inline).

### Step 15 — Self-check

After writing, verify:

- All required sections present (per the template — remember `statix` differences).
- "What's already in place" preamble is populated and cites at least one prior `IMPLEMENTED/*` spec (or, if backfill ran, the just-backfilled MVP spec).
- `ACCEPTANCE_CRITERIA` is present, sectioned-numbered, every criterion has tag + pre-conditions + prose.
- No template placeholder text survives.
- Frontmatter is valid YAML, `parent_spec` resolves to an existing file.
- All `[OPEN: ...]` markers are intentional.

If any check fails, fix it. If a fix isn't possible (e.g. no `IMPLEMENTED/*` to anchor preamble), surface the failure to the user.

### Step 16 — Output to user

In chat:

1. Confirm the file path where the spec was written.
2. Note the `parent_spec` it points at.
3. List any `[OPEN: <question>]` markers.
4. If backfill ran, mention the backfilled MVP spec path.
5. Suggest the next step: `/mvp-specification` (if scope-cutting wanted) or `/build-from-spec` (if the feature is small enough to ship as-is).

Don't ask "anything else?" Don't add a closing line.

## What this skill does not do

- **Does not run on claude.ai chat.** Claude Code only.
- **Does not handle project mode.** Sibling skill `/detailed-specification` is the project-mode entry point.
- **Does not modify existing implemented specs.** Read-only against `IMPLEMENTED/*.md`. Even backfill writes a *new* spec rather than editing existing files.
- **Does not invoke `/interview-me` automatically.** Suggests it when input is too thin; control returns to the user.
- **Does not write the MVP spec for the feature.** That's `/mvp-specification`'s job (when invoked — it's optional).
- **Does not auto-suggest DEFERRED items.** Only mentions them when the user references them (per the three-signal rule above).
- **Does not auto-fall-through to project mode.** When invoked against a fresh repo, asks the user whether they meant `/detailed-specification`.

## Frontmatter contract (produced)

The frontmatter this skill writes:

| Field | Value |
|---|---|
| `spec_type` | `"detailed"` |
| `mode` | `"feature"` |
| `name` | The feature name (uppercase snake_case) |
| `date_started` | The date the spec was written (YYYY-MM-DD) |
| `template` | The repo's actual template (detected via `CLAUDE.md`) |
| `status` | `"NOT_YET_IMPLEMENTED"` |
| `parent_spec` | Relative path to the most relevant `IMPLEMENTED_DETAILED_*.md`, falling back to `IMPLEMENTED_MVP_*.md` |

Downstream skills validate these fields. `parent_spec` non-empty is the unique signature of a feature-mode spec; project-mode detailed specs leave it empty.

## Examples

### Example 1 — clean feature extension, no backfill

User invokes `/extend-features` in a kitchen-sink-ts repo with `IMPLEMENTED/IMPLEMENTED_MVP_20260301_SPEC.md` (PageMark v1, the original project MVP). Pastes a description: "Add team accounts so users can share entry collections."

Skill reads template, detects feature mode, doesn't trigger backfill (clear IMPLEMENTED MVP exists). Loads the relevant prior spec (PageMark v1) for context. Cross-checks: PageMark v1 had "any multi-user / collaboration feature" in Out of scope. Surfaces this to the user: "Original MVP marked multi-user as out-of-scope. Continuing past that boundary — confirm?" User confirms.

Asks 6 follow-up questions (much lower than project-mode's 15-20 average) — schema for team membership, how invites work, whether existing entries can be moved to a team, etc.

Writes `SPECIFICATIONS/NOT_YET_IMPLEMENTED/DETAILED_TEAM_ACCOUNTS_20260501.md`:
- Frontmatter: `parent_spec: "../IMPLEMENTED/IMPLEMENTED_MVP_20260301_SPEC.md"` (no IMPLEMENTED detailed spec exists, so falls back to MVP).
- "What's already in place" preamble references the PageMark v1 entries flow.
- AC section with ~12 criteria covering team creation, invite flow, entry sharing, permissions.

Output: file path, parent_spec, no `[OPEN]` markers, suggested next step (`/mvp-specification` or `/build-from-spec`).

### Example 2 — backfill triggered

User invokes `/extend-features` in a repo that's been worked on for months but has an empty `SPECIFICATIONS/IMPLEMENTED/`. The repo has 47 commits, custom routes, custom DB tables, customized auth. Asks: "Add Stripe billing."

Skill detects backfill needed (commit count, file-map diff, schema diff, empty IMPLEMENTED). Scans the repo, reconstructs the implemented MVP, presents the draft. User reviews, asks the skill to split into two phases (auth + dashboard, then later subscription scaffolding that was added but never completed). Skill writes two backfilled specs to `IMPLEMENTED/`.

Then proceeds with the actual feature spec for Stripe billing, with `parent_spec` pointing at the more-recent backfilled spec (the subscription-scaffolding one) since it's the more relevant anchor.

### Example 3 — refusal-equivalent: ambiguous mode

User invokes `/extend-features` in a fresh kitchen-sink-ts clone (just-cloned, only the template scaffold present, no `IMPLEMENTED/*`). Says: "I want to build a recipe tracker."

Skill's mode detection: no signs of prior work, sounds like a brand-new project description. Asks the user inline: "This looks like a new project rather than an extension to existing work. Want me to defer to `/detailed-specification` (project mode) instead?" User confirms project mode.

Skill exits, telling the user to run `/detailed-specification`. Doesn't auto-invoke.

### Example 4 — DEFERRED file referenced

User invokes `/extend-features` and mentions: "Pick up the search filtering thing from the deferred list."

Skill detects "deferred list" reference. Loads `MVP_*_DEFERRED.md` files in `NOT_YET_IMPLEMENTED/`. Finds an entry for "tag multi-select filtering" deferred from PageMark v1. Asks: "Is this what you meant? `MVP_20260301_DEFERRED.md` lists 'tag multi-select filtering: defer until the loop is proven'. Use as the basis for this feature spec?"

User confirms. Skill writes the feature spec, citing the deferred item in the "What's already in place" preamble (i.e., this was originally deferred and is now being resurrected).
