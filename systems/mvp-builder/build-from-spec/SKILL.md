---
name: build-from-spec
description: Use this skill when the user wants to implement a specification end-to-end against the live repo. Trigger on phrases like "build it", "implement this spec", "go through the acceptance criteria", "run the build", "execute the spec". Consumes any spec from `SPECIFICATIONS/NOT_YET_IMPLEMENTED/` that carries an `ACCEPTANCE_CRITERIA` section — detailed (project|feature) or MVP (project|feature). Runs a sequential criterion-by-criterion loop, committing per criterion on success and rolling back via `git reset` on failed attempts. Claude Code only — this skill writes code, runs commands, generates migrations, and makes commits. Designed for autonomous operation where possible. Refuses if the working tree is dirty or the user is not on `main`.
---

# build-from-spec

This skill consumes a specification carrying acceptance criteria and implements it against the live repo. One commit per acceptance criterion, sequential execution, per-attempt rollback via `git reset`, and an end-of-build report.

The input spec can be a detailed spec (project or feature mode) or an MVP spec (project or feature mode). The skill keys on the presence of an `ACCEPTANCE_CRITERIA` section, not the filename prefix. This means `/mvp-specification` is optional — you can flow detailed → build directly when scope-cutting isn't needed, or detailed → mvp → build when it is.

The skill runs in the context of a freshly-cloned repo (project mode) or an existing implemented repo (feature mode). It always reads the live template's `CLAUDE.md` to enforce the Principles section and use the agent-writable Reference section as guidance.

The bias is toward **autonomy where possible, transparency where not**. The skill makes its own judgment calls about implementation choices (templates' recipes are guidance, not prescription) and uses dependency analysis to decide what to skip after a failure. Mid-build interactions are minimal — max 5 questions per batch — and only when genuinely blocked.

## Boundaries

- **Claude Code only.** This skill executes code, runs commands, generates migrations, and writes commits. There is no claude.ai surface for it.
- **Both modes.** Project mode and feature mode. Same workflow with mode-specific differences in post-build artifact updates.
- **Any spec with AC.** Accepts detailed specs (project or feature mode) and MVP specs (project or feature mode). The skill keys on the `ACCEPTANCE_CRITERIA` section being present and well-formed, not on the filename prefix.
- **Re-runnable.** This skill is on the critical path for full builds, extension, and fixing previously-failed criteria. The same invocation handles all three — it detects already-passed criteria via git history and decides what to attempt.
- **Sequential per criterion.** No parallel execution. Criteria run in spec order (sectioned numbering: 1.1, 1.2, 2.1, …).
- **Trust the upstream.** The spec was produced by `/detailed-specification`, `/extend-features`, or `/mvp-specification`. Don't re-validate the AC contract structure — only check that `ACCEPTANCE_CRITERIA` and the spec body are present.
- **No state file.** The git history is the progress log. The skill greps commit messages to detect already-passed criteria.
- **Persistent verification wherever possible.** Each criterion's verification is written as a runnable artifact (test file, fixture-based assertion) that anyone can re-run later. Transient verification only when the criterion can't be persisted (most `[USER_VERIFIES]` cases).
- **Enforces template Principles.** The Principles section of `CLAUDE.md` (changes are surgical, no speculative abstractions, lazy-init, etc.) is policy. The agent-writable Reference section is guidance.

## Workflow

### Step 1 — Pre-flight repo state

Validate the working tree:

- The user is on `main`.
- The working tree is clean (no uncommitted changes, no untracked files that would be touched by the build).

If either fails, **refuse**. Don't try to stash, switch branches, or otherwise mutate state. The user resolves their own working state.

### Step 2 — Locate and validate the spec

Look in `SPECIFICATIONS/NOT_YET_IMPLEMENTED/` for any `*.md` file that carries an `ACCEPTANCE_CRITERIA` section. Acceptable inputs:

- `MVP_{YYYYMMDD}_SPEC.md` — project-mode MVP spec.
- `MVP_{FEATURE_NAME}_{YYYYMMDD}.md` — feature-mode MVP spec.
- `DETAILED_{PROJECT_NAME}_{YYYYMMDD}.md` — project-mode detailed spec (when `/mvp-specification` was bypassed).
- `DETAILED_{FEATURE_NAME}_{YYYYMMDD}.md` — feature-mode detailed spec (when `/mvp-specification` was bypassed).

**Refusal cases:**

- **No spec with AC found.** Refuse. Suggest `/detailed-specification`, `/extend-features`, or `/mvp-specification` depending on what the user is trying to do.
- **Multiple specs that look like duplicates** (same project, or same feature name across days). Refuse. Surface the candidates to the user and ask which one to build.
- **Single spec found** with AC. Proceed with it.
- **Multiple distinct specs** (different feature names, or detailed + MVP for the same project). Ask the user which one to build. If both detailed and MVP exist for the same scope, prefer the MVP (it's the tighter cut) unless the user says otherwise.

Validate the spec contains:

- An `ACCEPTANCE_CRITERIA` section with at least one criterion.
- Frontmatter with `spec_type` (`detailed` or `mvp`), `mode`, `name`, `template`.

Don't deeply re-validate the AC contract (numbering format, pre-condition presence, etc.). Trust that the upstream skill produced it correctly. The spec body provides build context.

#### Allowed-but-warned: every criterion is `[USER_VERIFIES]`

If every criterion in the spec is tagged `[USER_VERIFIES]`, proceed but warn the user that the build will have no automated verification — the feedback loop will be slow and every criterion will require manual review.

### Step 3 — Verify template match

Read the repo's `CLAUDE.md` to identify the template (the first heading typically declares it: `# kitchen-sink-ts — Ways of Working`).

Compare against the spec's frontmatter `template`.

- **Match.** Proceed silently.
- **Mismatch.** Ask the user. If the mismatch materially changes how the build would proceed (e.g. spec assumes auth scaffold but the repo is statix), push back to the upstream skill (`/mvp-specification` or `/extend-features`) — the spec needs revision before build, not a build-time fudge. If the mismatch is benign (spec was written assuming X, but the actual repo is structurally compatible), proceed with explicit user confirmation.

### Step 4 — Detect mode

Read `mode` from the spec frontmatter.

- `mode: project` — full project build. Post-build will (re)generate the full agent-writable Reference section in `CLAUDE.md`.
- `mode: feature` — feature additions. Post-build will only update what's necessary (delta append to Reference, not full regen). Don't touch `DATABASEMODEL.md` if no schema change. Don't touch `README.md` if no public-surface change.

### Step 5 — Load template context

Load **once-per-build** into working memory:

- The full input spec (frontmatter + body + ACCEPTANCE_CRITERIA).
- The template's `CLAUDE.md` — both the Principles section (above the divider) and the agent-writable Reference section (below).
- `DATABASEMODEL.md` if it exists (kitchen-sink-ts and twotier; statix has none).

The Principles section is policy. The Reference section is guidance — recipes, file maps, and golden paths describe known-good shapes for common operations, but adapt as needed. Don't follow recipes verbatim if the criterion calls for something different.

Load **on-demand** when a criterion explicitly references prior implemented work:

- `SPECIFICATIONS/IMPLEMENTED/IMPLEMENTED_*.md` — only if a criterion's text or pre-conditions reference an existing implemented behavior.

### Step 6 — Build the dependency graph

Walk the spec's `ACCEPTANCE_CRITERIA` and look for explicit pre-condition references between criteria, e.g. `Pre-conditions: Given criterion 1.1 has succeeded, …`.

Build a directed graph: criterion N depends on criterion M if N's pre-conditions explicitly reference M.

This graph drives downstream skipping. After a criterion fails, any criterion with a transitive explicit dependency on it is **skipped as "blocked by upstream"** — not attempted, not failed. Independent criteria continue.

Implicit dependencies (criterion N implicitly relies on criterion M because N's section logically depends on M's section) are NOT inferred. Only explicit references count. If the user wanted a dependency, they wrote it into the pre-conditions.

### Step 7 — Detect already-passed criteria

Grep the git log for commits matching the convention `feat(<scope>): criterion <N.M> — <title>` (see Step 8 for the convention).

For each already-passed criterion:

- **If the criterion has a persistent verification artifact** (test file, fixture-based assertion in the repo): re-run the verification. If it still passes, mark passed and skip implementation. If it now fails (regression), surface to the user and ask: re-implement, accept the regression, or abort.
- **If the criterion has no persistent verification** (USER_VERIFIES, or some other transient case): trust the prior commit. Mark passed and skip. Do not re-attempt.

This makes `/build-from-spec` safe to re-run after a partial success — it picks up from where it left off without re-doing finished work.

### Step 8 — Per-criterion loop

Iterate through criteria in spec order (sectioned numbering: 1.1 → 1.2 → 2.1 → 2.2 → …).

For each criterion:

1. **Check dependency status.** If any explicit upstream dependency failed, mark this criterion `BLOCKED_BY_UPSTREAM` and skip. Continue to the next criterion.

2. **Check already-passed status** (from Step 7). If passed, skip.

3. **Set attempt budget by tag:**
   - `[BLOCKING]` (explicit or default) → up to 3 attempts.
   - `[NICE_TO_HAVE]` → 1 attempt only.
   - `[USER_VERIFIES]` → 1 attempt that ends with the criterion in the report queue (see USER_VERIFIES flow below); does not block subsequent criteria.

4. **Attempt loop.** For each attempt within budget:

   a. **Plan.** Load: the criterion text, pre-conditions, the spec section it lives in, and the diff of the just-completed prior criterion (if any). Decide:
      - What code needs to change (files, functions, schema).
      - What the verification mechanism is (heuristic-picked from the criterion text — see "Verification mechanism heuristics" below).
      - What the persistent verification artifact will be (test file path, fixture name).

   b. **Implement.** Write the code. Follow the templates' Principles (surgical, no speculative abstractions, lazy-init for env-dependent SDKs, etc.).

   c. **Migrations.** If the implementation requires a DB schema change:
      - Generate the up-migration via the template's mechanism (`bun run db:generate` for kitchen-sink-ts; `bun run --filter './server' db:generate` for twotier).
      - **Generate a paired down-migration** as a rule. The down-migration must reverse the up-migration. Without it, this attempt cannot be rolled back cleanly.
      - Apply the up-migration.

   d. **Write the verification artifact** (persistent test file, fixture-based assertion).

   e. **Run the verification.**

   f. **On success:**
      - Stage all changes (code + migration files + verification artifact).
      - Commit with the convention: `feat(<scope>): criterion <N.M> — <criterion title>`. The scope is the section name (`auth`, `entries`, `subscription`, etc., lowercased).
      - Mark criterion `PASSED`. Continue to the next criterion.

   g. **On failure:**
      - **Roll back the migration first** if one was applied (run the paired down-migration).
      - `git reset --hard HEAD` to discard all uncommitted changes.
      - Increment attempt counter.
      - If attempts remaining for this criterion's tag, retry from step 4a (next attempt). Use the failure mode from this attempt as additional context for the next plan — don't repeat the same approach blindly.
      - If attempts exhausted, mark criterion `FAILED`. Continue to the next criterion (the dependency graph from Step 6 may cause downstream criteria to skip; that's expected).

#### Verification mechanism heuristics

Pick from the menu in this template's `CLAUDE.md`:

- Criterion text mentions HTTP routes / endpoints / status codes / Zod → **HTTP+Zod** (kitchen-sink-ts, twotier). Use the test runner's in-process server fixture; do not spin up `bun run dev`.
- Criterion text mentions DB rows / queries / schema → **cmd-exit-0** via Vitest hitting the DB through the service layer.
- Criterion text mentions a CLI command's output → **cmd-exit-0** wrapping the command in a test.
- Criterion text mentions a built page's DOM (statix) → **DOM assertion** (built-page check).
- Criterion text describes visual / typographic / content-tone behavior, OR the criterion is tagged `[USER_VERIFIES]` → **USER_VERIFIES** flow.
- Criterion text mentions linting / typechecking / formatting → **cmd-exit-0** wrapping the existing template script (`bun run check`, `bun run typecheck`).

When the criterion text supports multiple mechanisms, prefer persistent (test file) over transient (one-off curl). Prefer fixture-based (in-process) over external-process (running dev server).

#### `[USER_VERIFIES]` flow

`[USER_VERIFIES]` criteria can't be machine-asserted, but the build is designed for autonomous operation — it does **not** pause synchronously waiting for user eyeballs.

For each `[USER_VERIFIES]` criterion:

1. Implement the behavior (code change, no separate verification artifact since there's nothing automated to write).
2. Commit with a marker in the message: `feat(<scope>): criterion <N.M> — <title> (USER_VERIFIES, pending review)`.
3. Add the criterion to the end-of-build report's "needs human review" queue.
4. Continue to the next criterion. Do not block.

The user reviews the implementation offline. Iteration on USER_VERIFIES criteria (the user wants something fixed) happens via a separate `/build-from-spec` re-run — for v1, that re-run treats the criterion as "already attempted" and the user must explicitly request re-implementation. Async signal-based re-attempts are out of scope for v1.

### Step 9 — Mid-build interactions

The skill is allowed to ask the user mid-build, but **iterative and minimal**:

- Maximum **5 questions per batch.**
- Only when genuinely blocked (ambiguous spec wording, irreconcilable mismatch with template, decision needed before proceeding).
- Never to second-guess the spec — if the spec says X, do X. Push back via refusal, not via interrogation.
- Surface the question, wait for the answer, proceed. Don't write Q&A files for build-time questions; they're conversational.

### Step 10 — Post-build artifact updates

After the per-criterion loop completes (all criteria attempted, passed, failed, or blocked), update the post-build artifacts.

**Project mode:**

- Regenerate the full agent-writable Reference section in `CLAUDE.md` (the section below `<!-- AGENT-WRITABLE BELOW -->`). File map, recipes, golden paths reflect the now-implemented project state.
- Update `README.md` if the project introduces new public-surface things (endpoints, env vars, scripts) beyond what the template already documents.
- Update `DATABASEMODEL.md` if the project added or changed tables. Use `bun run db:doc` (or the template-equivalent) to seed a mermaid skeleton, then hand-edit prose.

**Feature mode:**

- Append delta updates to the agent-writable Reference section — don't regenerate full. New file map entries, new recipes specific to the feature, new golden paths.
- Update `README.md` only if the feature introduces new public-surface things.
- Update `DATABASEMODEL.md` only if the feature changed schema.
- Don't touch sections of `CLAUDE.md` that the feature didn't affect.

Each artifact update is its own commit, messaged conventionally:

- `docs(claude): regenerate Reference section for <project name>` (project) or `docs(claude): add Reference entries for <feature name>` (feature).
- `docs(readme): add <thing>` if README changed.
- `docs(dbmodel): add <table list>` if DATABASEMODEL changed.

### Step 11 — Spec lifecycle

**Only if at least one criterion passed**, move the spec from `NOT_YET_IMPLEMENTED/` to `IMPLEMENTED/` with the `IMPLEMENTED_` prefix in a separate commit (after the implementation commits and the artifact-update commits).

- File: `SPECIFICATIONS/NOT_YET_IMPLEMENTED/<spec>.md` → `SPECIFICATIONS/IMPLEMENTED/IMPLEMENTED_<spec>.md`. The `IMPLEMENTED_` prefix is added regardless of whether the input was a `DETAILED_` or `MVP_` spec.
- Commit message: `chore(specs): mark <spec name> as implemented`.

**If zero criteria passed** (total failure): the spec **stays in `NOT_YET_IMPLEMENTED/`**. Nothing was actually built; the lifecycle stays open for a future `/build-from-spec` re-run.

The DEFERRED sibling (`MVP_{YYYYMMDD}_DEFERRED.md` or `MVP_{FEATURE_NAME}_{YYYYMMDD}_DEFERRED.md`) **stays in `NOT_YET_IMPLEMENTED/` regardless**. It's not moved with the implemented spec. `/extend-features` reads it later to source candidate work.

### Step 12 — Write the end-of-build report

Write `AGENT_REPORTS/BUILD_REPORT_{YYYYMMDD}_{HHMM}.md` with:

```yaml
---
report_type: "build"
spec_consumed: "<spec filename>"
mode: "<project | feature>"
template: "<template name>"
build_started: "<ISO 8601 timestamp>"
build_completed: "<ISO 8601 timestamp>"
result_summary:
  total_criteria: <N>
  passed: <N>
  failed: <N>
  blocked_by_upstream: <N>
  pending_user_review: <N>
---
```

Body sections:

- **Summary** — one paragraph: what was attempted, what passed, what failed, what's pending review.
- **Per-criterion results** — table: number, title, tag, status (PASSED / FAILED / BLOCKED_BY_UPSTREAM / PENDING_USER_REVIEW), commit SHA (if PASSED or PENDING_USER_REVIEW), failure summary (if FAILED).
- **Pending user review** — list of `[USER_VERIFIES]` criteria with: commit SHA, what the user should look at (file paths, URL if applicable), and what specifically to verify.
- **Failures** — for each `FAILED` criterion: attempt count, what was tried each attempt, why each attempt failed.
- **Skipped (blocked by upstream)** — for each `BLOCKED_BY_UPSTREAM` criterion: which upstream criterion's failure caused the skip.
- **Suggested next actions** — what the user might do (re-run after fixing X, manually verify Y, accept Z as deferred).

**Do not commit this file.** The user commits it (or doesn't) per their preference. The `AGENT_REPORTS/` directory should exist or be created — but not committed by this skill. (If the directory needs to be `.gitignore`d, that's a user decision.)

The report exists to support future HITL integrations (Discord posts, ticket creation, async review surfaces).

### Step 13 — Final summary to user

In the chat output:

- One sentence: how many criteria attempted, how many passed, how many failed, how many pending review.
- The path to the report file.
- The spec's new location (`IMPLEMENTED/IMPLEMENTED_<spec>.md` if ≥1 passed; still in `NOT_YET_IMPLEMENTED/` if zero).
- Any criteria pending user review, with the commit SHAs (so the user can `git show` them).
- Don't dump the full report into chat — the file is the deliverable.

Don't ask "anything else?" Don't add a closing line.

## What this skill does not do

- **Does not run on claude.ai chat.** Claude Code only.
- **Does not invoke other skills.** No auto-suggesting `/railway-deployment`. No auto-fall-through to `/mvp-specification` or the upstream spec skills if a spec is missing — refuse and tell the user to run the right upstream skill.
- **Does not modify the spec.** The spec is read-only input. Move it through the lifecycle, but don't edit its content.
- **Does not re-implement passed criteria** unless the user explicitly asks. Re-runs detect prior passes via git history and skip them (or re-verify if a persistent artifact exists).
- **Does not pause synchronously on `[USER_VERIFIES]`.** Implements, commits, queues for review, continues.
- **Does not generate up-migrations without paired down-migrations.** The down-migration is part of the attempt's contract; without it, rollback can't restore state.
- **Does not stash, branch, or rebase to deal with a dirty working tree.** Refuses upfront.
- **Does not move the DEFERRED sibling.** Stays in `NOT_YET_IMPLEMENTED/` for `/extend-features` to consume later.

## Frontmatter contract (consumed)

The input spec frontmatter the skill reads:

| Field | Used for |
|---|---|
| `spec_type` | Must be `"detailed"` or `"mvp"`. Refuse otherwise. |
| `mode` | Drives Step 4 (post-build artifact policy). |
| `name` | Used in commit messages and the build report. |
| `date_started` | Surfaced in the build report. |
| `template` | Compared against the repo's actual template (Step 3). |
| `status` | Should be `"NOT_YET_IMPLEMENTED"` on input. |
| `parent_spec` | Read for context but not validated by this skill. |

## Commit message convention

Per criterion, on success:

```
feat(<scope>): criterion <N.M> — <criterion title>
```

Examples:

- `feat(auth): criterion 1.1 — sign in with valid creds`
- `feat(entries): criterion 2.3 — reject empty title`
- `feat(billing): criterion 4.2 — subscription webhook persists status`

For `[USER_VERIFIES]` criteria, append `(USER_VERIFIES, pending review)`:

- `feat(dashboard): criterion 5.1 — entries list visual order (USER_VERIFIES, pending review)`

Post-build artifact commits:

- `docs(claude): regenerate Reference section for <project name>` (project mode)
- `docs(claude): add Reference entries for <feature name>` (feature mode)
- `docs(readme): add <thing>` (only if changed)
- `docs(dbmodel): add <table list>` (only if changed)
- `chore(specs): mark <spec name> as implemented`

The `criterion N.M` token is grep-able. Step 7 uses it to detect already-passed criteria on re-run.

## Examples

### Example 1 — clean project-mode build

User invokes `/build-from-spec` in a freshly-cloned `kitchen-sink-ts` repo. `MVP_20260501_SPEC.md` exists in `NOT_YET_IMPLEMENTED/` with 17 criteria across 5 sections (auth, capture, browse-list, today's review, summarize). All `[BLOCKING]`, three `[USER_VERIFIES]`.

Skill runs: clean tree ✓, on main ✓, single MVP spec ✓, template matches ✓.

Loops through criteria. 14 pass first-attempt, 2 pass second-attempt, 1 fails after 3 attempts (criterion 4.2 — "today's review entry caches per user-day"; the agent couldn't get the cache idempotency right). 3 USER_VERIFIES criteria are implemented + flagged.

Post-build: regenerates `CLAUDE.md` Reference, updates `DATABASEMODEL.md` (two new tables), updates `README.md` (new env var for OpenRouter default model). Spec moved to `IMPLEMENTED/IMPLEMENTED_MVP_20260501_SPEC.md`. Report written to `AGENT_REPORTS/BUILD_REPORT_20260501_1432.md`.

Chat: "16 of 17 criteria passed (3 pending user review). 1 failed: criterion 4.2. Spec moved to IMPLEMENTED. Report at AGENT_REPORTS/BUILD_REPORT_20260501_1432.md."

### Example 2 — refusal: dirty tree

User invokes `/build-from-spec`. Working tree has uncommitted changes in `src/lib/db/schema.ts`.

Skill: refuses in two paragraphs. First: "Working tree has uncommitted changes in src/lib/db/schema.ts. /build-mvp requires a clean tree on main so per-attempt rollback works." Second: "Commit or stash your changes, then re-run."

### Example 3 — re-run after partial success

User invokes `/build-from-spec` again on the same repo as Example 1. Spec is now in `IMPLEMENTED/`, but the user wants to re-attempt criterion 4.2 (which previously failed). They first move the spec back to `NOT_YET_IMPLEMENTED/` manually.

Skill runs Step 7: greps the log, finds `criterion 1.1` through `criterion 4.1` and `criterion 4.3` through `criterion 5.4` as prior commits. Re-verifies the persistent ones (most pass). Marks them passed. Identifies criterion 4.2 as the only un-attempted-this-build criterion in the un-skipped graph.

Implements 4.2 (this time with a different cache strategy — uses the prior failed-attempt context). Passes on attempt 2. Commits, regenerates `CLAUDE.md` Reference (delta — only the changes), moves spec back to `IMPLEMENTED/`.

### Example 4 — feature mode with cross-spec contradiction

User invokes `/build-from-spec` in a repo that already has `IMPLEMENTED/IMPLEMENTED_MVP_20260301_SPEC.md` (PageMark v1). `NOT_YET_IMPLEMENTED/` has `MVP_PAGEMARK_MONETIZATION_20260501.md` (a feature spec re-enabling Polar).

Skill detects feature mode from the spec's `mode: feature`. Reads `IMPLEMENTED_MVP_20260301_SPEC.md` because criterion 2.1 of the new spec references "the existing entry list view". Detects the new spec requires Polar to be enabled (was REMOVE in the original). Implements env wiring, product setup, checkout, webhook, gating.

Post-build: appends Reference section delta in `CLAUDE.md` (new file map entries for billing routes), doesn't touch sections about entries (unchanged). Updates `README.md` (new POLAR_* env vars). Touches `DATABASEMODEL.md` (subscriptions table activation). Spec moves to `IMPLEMENTED/`.

### Example 5 — total failure

User invokes `/build-from-spec` on a spec where every criterion fails after 3 attempts (the spec is poorly aligned with what the template can support — assumes capabilities that don't exist).

Skill loops, every criterion `FAILED`. **Spec stays in `NOT_YET_IMPLEMENTED/`** — nothing was built. Post-build artifact updates skipped (nothing to document). Report written with detailed failure summaries per criterion.

Chat: "0 of 12 criteria passed. Spec stays in NOT_YET_IMPLEMENTED. Report at AGENT_REPORTS/BUILD_REPORT_20260501_0945.md. The spec likely needs revision before re-running — common failure pattern: <summary>."
