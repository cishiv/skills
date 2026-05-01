---
name: mvp-specification
description: Use this skill when the user wants to take an upstream detailed/extend specification and cut it down to the smallest valuable thing to ship — the MVP (minimum loveable product). Trigger on phrases like "cut scope", "what's the MVP here", "make this an MVP", "scope this down", "add acceptance criteria", "let's MVP this". Produces an `MVP_*.md` artifact (and an optional `MVP_*_DEFERRED.md` sibling) with EARS-flavored prose acceptance criteria. Handles both project mode (consumes a `DETAILED_*` spec) and feature mode (consumes an extend-features output). Runs on both Claude Code and claude.ai chat surfaces. Refuses without an upstream spec — push the user to `/detailed-specification` or `/extend-features` first.
---

# mvp-specification

This skill takes an upstream detailed (or extend) specification and produces the **MVP specification** — the smallest version that proves the product loop and delivers value to a real end user.

The upstream spec is the full vision; this skill cuts it. The MVP spec is the input to `/build-from-spec`, which loops on the `ACCEPTANCE_CRITERIA` until they pass.

The bias is toward **minimum loveable**, not minimum *viable*. Cuts are made on complexity and impact on extensibility, not on time. Templates already provide the heavy lifting — the cut should preserve everything the template gives for free and remove only the new work that doesn't earn its place.

## Boundaries

- **Optional in the pipeline.** This skill is no longer required between detailed-spec and build. `/build-from-spec` accepts detailed specs (which carry their own AC) directly. Invoke `/mvp-specification` when scope-cutting is wanted — when the detailed spec is too ambitious to ship as one piece, when a `MVP_*_DEFERRED.md` sibling is useful for downstream `/extend-features` consumption, or when the AC need a tighter slice than the full vision. Skip otherwise.
- **Both modes.** Project mode consumes a `DETAILED_*` spec from `/detailed-specification`. Feature mode consumes an extend output from `/extend-features`. The two upstream artifacts are structurally identical; only frontmatter (`mode`, `parent_spec`) and filename diverge.
- **Both surfaces.** Claude Code with the target repo on disk, and claude.ai chat without a repo. The skill detects which and adapts validation, output location, and template-knowledge sourcing.
- **Requires an upstream spec.** No upstream → refuse and suggest the upstream skill. Don't elicit the upstream spec inline; don't auto-fall-through.
- **Prose acceptance criteria.** Each criterion is human-readable EARS-flavored prose with first-class pre-conditions. The build agent figures out *how* to verify; this skill writes *what* must be true.
- **Scope-cutting is a conversation, not an algorithm.** Skill proposes cuts; user confirms. The skill stays mechanical (complexity, extensibility, MLP loop alignment) — it doesn't quantify in time or cost.
- **Template-aware throughout.** The skill must know what the chosen template provides for free before it proposes cuts or writes criteria. Things the template already does don't become AC; things the template provides but require configuration become CONFIGURE-flavored AC.
- **No silent decisions.** Every cut, every retained item, every criterion traces to either the upstream spec, a follow-up answer, or an explicit user-approved choice.

## Workflow

### Step 1 — Determine surface

The skill runs on one of two surfaces. Apply silently to subsequent steps; don't announce.

**Claude Code mode** — true if all of:
- The current working directory contains a `SPECIFICATIONS/` directory, OR can walk up from the cwd to find one within 3 levels.
- Filesystem write tools are available.

In this mode, the skill writes the MVP spec (and DEFERRED sibling, if any) directly into the target repo.

**Not Claude Code mode** — anything else. claude.ai chat, mobile, web, sandboxed environments without a target repo.

In this mode, the skill produces `.md` artifacts rendered inline. If Obsidian MCP is connected, it also silently mirrors to `AI/Skill Systems/MVP_SYSTEM/STAGING/` for async hand-off.

### Step 2 — Locate and read the upstream spec

**Always ask the user explicitly which upstream spec to consume.** Acceptable forms:

- A path to a `DETAILED_*.md` or extend output in `SPECIFICATIONS/NOT_YET_IMPLEMENTED/`
- The full upstream spec pasted into chat
- An Obsidian path to a staged spec from a prior `/detailed-specification` claude.ai run

If the user invokes the skill without naming a spec and there's an obvious candidate (a single `DETAILED_*.md` in `NOT_YET_IMPLEMENTED/` modified in the last hour), propose it. Otherwise ask.

Validate the upstream spec's frontmatter shape:

| Field | Expected |
|---|---|
| `spec_type` | `"detailed"` (project mode) or whatever `/extend-features` writes (feature mode) |
| `mode` | `"project"` or `"feature"` |
| `name` | non-empty |
| `template` | one of `"kitchen-sink-ts"`, `"kitchen-sink-twotier"`, `"statix"` |
| `status` | `"NOT_YET_IMPLEMENTED"` |
| `parent_spec` | empty for project mode, populated for feature mode |

#### Refusal: no upstream spec

If the user invokes `/mvp-specification` without an upstream spec — refuse in two paragraphs. First paragraph: state that this skill consumes an existing detailed/extend spec and can't operate without one. Second paragraph: suggest running `/detailed-specification` (project mode) or `/extend-features` (feature mode) first, depending on whether this is a new repo or a feature on an existing one.

#### Refusal: upstream spec has wrong shape

Frontmatter is missing required fields, or `status` is already `IMPLEMENTED`, or the spec_type doesn't match either project or feature pattern. Refuse with the specific shape error and stop.

### Step 3 — Determine mode

Read `mode` from the upstream frontmatter. That's the source of truth.

**Fallback chain if `mode` is missing or unreadable:**

1. Ask the user directly: "Is this a new project or a feature addition?"
2. Infer from repo state: if `SPECIFICATIONS/IMPLEMENTED/MVP_*` exists, it's almost certainly feature mode. If the repo is bare, project mode.

Don't infer silently when frontmatter says one thing and repo state says another. Surface the conflict.

### Step 4 — Verify template match

The upstream frontmatter declares a `template`. Cross-check it against the actual repo:

- **Claude Code mode:** read the repo's `CLAUDE.md` or `package.json` to identify the template. If it matches, proceed silently. If it doesn't, ask the user — don't override.
- **Not Claude Code mode:** trust the frontmatter and proceed. The repo isn't reachable for cross-check.

If the user wants to override a mismatch, get an explicit confirmation. Don't proceed on assumption.

### Step 5 — Read template capabilities

Before any scope-cutting or criterion-writing, the skill must know what the chosen template provides for free. Sources, in order:

1. **Live read (Claude Code mode with repo on disk).** Read the repo's `CLAUDE.md` (especially the agent-writable Reference section), `DATABASEMODEL.md` if present, and the scaffold tree under `src/`. This is the source of truth.
2. **GitHub MCP.** If connected, fetch `CLAUDE.md` from the template repo's `main` branch.
3. **`web_fetch`.** Fetch the raw URL: `https://raw.githubusercontent.com/cishiv/<template>/main/CLAUDE.md`.
4. **Bundled snapshot.** Read `references/template-capabilities/<template>.md` from this skill's own bundled references. Each snapshot summarizes what the template provides for free, what's not included, file map essentials, and the dev loop.

If steps 1–3 fail and the bundled snapshot is used, **explicitly tell the user**:
- Which snapshot date is in use.
- That the snapshot may be stale.
- A recommendation to verify the produced spec against the live template before committing.

The output of this step is a working mental model of the template's affordances. Internally, classify every capability into:

- **PROVIDED** — works out of the box (e.g. BetterAuth scaffold for kitchen-sink-ts, content pipeline for statix).
- **CONFIGURABLE** — present in scaffold but requires wiring (env vars, route registration, schema additions).
- **NOT_PRESENT** — not in the template; must be built from scratch if needed.

This classification drives the next steps.

### Step 6 — Identify MVP-relevant `[OPEN]` markers

Scan the upstream spec for `[OPEN: ...]` markers.

For each marker, decide whether it pertains to MVP scope. The skill makes a first-pass guess (based on which user flow / data field / integration the marker sits in) and surfaces them to the user.

- **MVP-relevant `[OPEN]`** — must be resolved inline before scope-cutting. The skill asks the question now; the resolution lives in the new MVP spec, not retroactively in the upstream spec.
- **Not-MVP-relevant `[OPEN]`** — ignore. They stay in the upstream spec untouched. They're not the MVP's problem.

If the user is unsure whether an `[OPEN]` is MVP-relevant, lean toward "MVP-relevant" and ask. Cheaper to resolve once than to discover mid-build.

### Step 7 — Cross-check (feature mode only)

Skip this step in project mode.

In feature mode, the skill must check the proposed feature against:

- All `IMPLEMENTED/MVP_*.md` and `IMPLEMENTED/DETAILED_*.md` files in the repo.
- The `Out of scope` section of every implemented spec.

Look for:

- **Direct contradictions.** The feature does X, but a prior implemented spec marked X as out-of-scope or implemented an incompatible behavior.
- **Overlapping scope.** The feature touches data or flows already owned by an implemented spec, in ways that would change the existing behavior.
- **Stale assumptions.** The upstream extend spec assumes integration Y exists, but the project's most recent implemented spec marked Y as REMOVE.

For each finding, surface to the user and ask. Don't proceed on contradiction.

### Step 8 — Scope cut

This is the heart of the skill. The goal: the **simplest version that proves the product loop AND lets a real end user get value from it.**

**Default mode is single-pass.** The skill proposes a complete cut in one round. If the user wants iterative refinement (cut, look at it, cut more), they can opt in by saying so.

**Skill proposes; user confirms.** The skill suggests cuts based on:

- **Complexity.** A feature that touches many parts of the system, requires coordination across data model + UI + integration + auth, etc. Higher complexity = stronger candidate for deferral.
- **Impact on extensibility.** Implementing a half-baked version now will lock in shape that's hard to undo. If the v1 implementation would constrain future work, defer rather than commit.
- **Template alignment.** If the template provides it for free or with light configuration, **keep it** (it's basically free). If the template doesn't provide it and the feature isn't load-bearing for the loop, **cut it**.
- **MLP loop alignment.** Does this feature contribute directly to the loop the product proves? If yes, keep. If it's auxiliary (analytics dashboards, advanced search, fancy UI states), defer.

**Stay mechanical.** Don't quantify in time or cost ("this would take 6 hours"). Frame in terms of complexity and extensibility ("this would require a custom job queue, which the template doesn't have, and would lock the data model into a job-aware shape").

**Dependency chains.** If the user wants to keep feature X but X depends on feature Y that they're cutting — surface the conflict explicitly. Then trust the user's resolution. Don't auto-include Y, don't refuse to cut Y.

**The MLP framing.** Every kept item must answer: "does this contribute to the loop a real end user gets value from?" Items that don't — even if they look small — should be cut.

#### Refusal: total cut

If the user wants to cut the entire spec down to nothing — refuse. There's no point producing an empty MVP. Tell the user the detailed spec needs at least one user-facing capability worth shipping; if nothing in the detailed spec qualifies, the issue is upstream.

### Step 9 — Write `ACCEPTANCE_CRITERIA`

Once the cut is settled, write the `ACCEPTANCE_CRITERIA` section.

#### Authoring style — EARS-flavored prose

Use **free-form prose inspired by EARS** (Easy Approach to Requirements Syntax). Don't enforce verbatim EARS patterns — the discipline matters more than the syntax.

EARS patterns to draw from:

- **Ubiquitous** — "The system shall <response>." Always-true requirements.
- **Event-driven** — "WHEN <event>, the system shall <response>."
- **State-driven** — "WHILE <state>, the system shall <response>."
- **Optional** — "WHERE <feature included>, the system shall <response>." Use for product-level optional features (not env-flag/feature-flag wiring, which is implementation detail).
- **Unwanted behavior** — "IF <unwanted condition>, THEN the system shall <response>." Use for error paths.

**Combined patterns are encouraged.** "WHEN a user submits a valid form, IF the email is unverified, THEN the system shall reject the request with a verification prompt." Combining patterns produces precise criteria; favor expressiveness over rigidity.

#### Structure per criterion

Each criterion has:

- **Number** — sectioned (auth: 1.1, 1.2; entries: 2.1, 2.2; etc.). Sections group by user flow or domain.
- **Tag** — `[BLOCKING]` or `[NICE_TO_HAVE]`. The skill recommends; the user decides. Default: items in primary user flows are `[BLOCKING]`; auxiliary polish is `[NICE_TO_HAVE]`.
- **Pre-conditions** — first-class field. State the world-state required before the criterion applies (e.g. "Given a logged-in user with at least one entry, …"). Pre-conditions can reference earlier criteria's success ("Given criterion 1.1 has succeeded, …") because `/build-from-spec` runs criteria in order.
- **Criterion text** — EARS-flavored prose.
- **Optional `[USER_VERIFIES]` tag** — for criteria that can't be machine-verified (e.g. "the dashboard shows three tiles in a visually pleasant order"). Mark these explicitly. The build agent will surface them to the user for manual sign-off.

Example shape:

```markdown
### 2. Entries

#### 2.1 Create entry [BLOCKING]

**Pre-conditions:** Given a logged-in user.

WHEN the user submits the new-entry form with a non-empty title and a valid `date_read`, the system shall persist the entry to the `entries` table and return the created row.

#### 2.2 Reject empty title [BLOCKING]

**Pre-conditions:** Given a logged-in user.

IF the new-entry form is submitted with an empty or whitespace-only title, THEN the system shall reject the request with a 400 status and a Zod issue describing the failed `title.min(1)` validation.

#### 2.3 List entries — visual order [NICE_TO_HAVE] [USER_VERIFIES]

**Pre-conditions:** Given a logged-in user with at least three entries.

WHEN the user opens the entries list, the entries shall appear in reverse-chronological order by `date_read`, with consistent vertical spacing and clear separation between entries.
```

#### NEW vs CONFIGURE vs PROVIDED

- **NEW criteria** — verify behavior the build agent must implement from scratch. Most criteria fall here.
- **CONFIGURE criteria** — verify that template-provided functionality is wired correctly for this project (e.g. "BetterAuth email/password is enabled and Polar/R2 are not initialized in the running app"). These are short, often single-line.
- **PROVIDED capabilities don't become criteria.** They're noted in the spec body's Architecture hints section so the build agent doesn't accidentally rebuild them.

#### Coverage discipline

For everything in the MVP, comprehensive error path coverage is required. The user reviews the spec before `/build-from-spec` runs, so over-specification is the right side to err on. If a happy path has a criterion, its primary error path needs one too.

UI-only behavioral assertions are allowed as `[USER_VERIFIES]` prose criteria.

#### No cap on count

Don't artificially trim criteria. `/build-from-spec` loops on each up to 3 attempts, so the count matters for build time — but completeness matters more. The user sees the spec before build runs.

### Step 10 — Pre-write validation

Before writing the MVP spec (and DEFERRED sibling), validate:

1. The target repo has a `SPECIFICATIONS/NOT_YET_IMPLEMENTED/` directory (Claude Code mode) or the Obsidian staging path is reachable (Not Claude Code mode).
2. No filename collision in the target location.

#### Same-day collision policy

If a `MVP_*.md` already exists for the same project/feature on the same day — **don't refuse hard like `/detailed-specification` does**. Surface to the user and ask what they want to do. From their perspective the new MVP might not be the same thing, even if the filename collides; let them decide whether to overwrite, rename (e.g. add a discriminator to `name`), or abort.

### Step 11 — Write the MVP spec (atomic with DEFERRED if applicable)

**Filename, by mode:**

- Project mode: `MVP_{YYYYMMDD}_SPEC.md`
- Feature mode: `MVP_{FEATURE_NAME}_{YYYYMMDD}.md`

**DEFERRED sibling, if anything is deferred:**

- Project mode: `MVP_{YYYYMMDD}_DEFERRED.md`
- Feature mode: `MVP_{FEATURE_NAME}_{YYYYMMDD}_DEFERRED.md`

**The MVP spec and DEFERRED file are written as an atomic pair.** Both succeed or neither. If the file system / Obsidian write fails on the second, undo the first.

Don't write a DEFERRED file if there's nothing deferred (i.e. the MVP equals the upstream detailed spec exactly, including the upstream's `Out of scope` section being empty).

**Output location, by surface:**

- **Claude Code mode:** write to `<repo>/SPECIFICATIONS/NOT_YET_IMPLEMENTED/<filename>` directly. The file in the repo is the canonical deliverable.
- **Not Claude Code mode:** the canonical deliverable is the `.md` artifact rendered inline in the response. If Obsidian MCP is connected, also silently write to `AI/Skill Systems/MVP_SYSTEM/STAGING/<filename>` as a backup.

**Frontmatter** (exactly this shape):

```yaml
---
spec_type: "mvp"
mode: "project"          # or "feature"
name: "<NAME>"           # carry forward from upstream
date_started: "<YYYY-MM-DD>"
template: "<chosen template>"
status: "NOT_YET_IMPLEMENTED"
parent_spec: "<upstream filename>"
---
```

`parent_spec` is the upstream spec's filename (e.g. `DETAILED_PAGEMARK_20260427.md`). Not a path, not the upstream `name` field. Filename so a reader can resolve it directly with `ls SPECIFICATIONS/NOT_YET_IMPLEMENTED/`.

**Sections (in this order):**

1. **Problem statement** — carry forward from upstream, possibly trimmed if the cut narrowed the problem.
2. **MVP scope statement** — one paragraph naming the loop being proven and the value the end user gets. This is the MLP framing made explicit.
3. **User flows** — only flows in the MVP. Flows entirely deferred go to the DEFERRED file.
4. **Data model changes** (or "Content model changes" for statix) — only entities/fields in the MVP. Deferred entities go to DEFERRED.
5. **Architecture hints** — including PROVIDED capabilities being relied on (so the build agent doesn't rebuild them) and any CONFIGURE actions needed.
6. **Integrations** (omit for statix) —
   - **Project mode:** copy the upstream spec's KEEP/REMOVE/NOT_PRESENT decisions verbatim. They're immutable at this stage.
   - **Feature mode:** carry forward but note any integrations newly required by this feature (e.g. "polar: was REMOVE in initial spec; this feature requires it — re-enable as part of build").
7. **`ACCEPTANCE_CRITERIA`** — see Step 9.
8. **Out of scope** — carry forward from upstream's Out of scope section verbatim. Items deferred from this MVP cycle go to the DEFERRED file, not here.

**DEFERRED file structure:**

```yaml
---
spec_type: "deferred"
mode: "project"          # or "feature"
name: "<NAME>"
date_started: "<YYYY-MM-DD>"
template: "<chosen template>"
parent_spec: "<MVP filename>"   # not the upstream detailed/extend spec
---
```

Body: list each deferred item with a one-sentence rationale. Group by source — "deferred from this MVP cycle" vs "carried from upstream Out of scope" — so future readers know the difference between *we considered and cut* and *was always out of scope*.

### Step 12 — Self-check

After writing, verify:

- All required sections present.
- No template placeholder text.
- Frontmatter is valid YAML and matches the contract.
- Every criterion has: number, tag, pre-conditions, criterion text. `[USER_VERIFIES]` only on criteria that genuinely can't be machine-asserted.
- Every criterion is in a numbered section (no flat top-level criteria).
- DEFERRED file exists if and only if there are deferred items.
- Atomic pair: both files exist (or neither, on rollback).

If any check fails, fix it. If a fix isn't possible (e.g. template mismatch surfaced late), undo the writes and surface to the user.

### Step 13 — Output to user

Output behavior depends on surface (per Step 1).

**Claude Code mode:**
1. Confirm the file paths written (MVP spec + DEFERRED if applicable).
2. List any unresolved decisions, BLOCKING criterion count, NICE_TO_HAVE count, USER_VERIFIES count.
3. Don't dump the full spec into chat — the file is the deliverable.

**Not Claude Code mode:**
1. The `.md` artifact rendered inline IS the deliverable. Don't repeat the content as plain chat text.
2. If a DEFERRED file was written, render it as a second artifact.
3. If Obsidian staging was used, mention the staging paths briefly.
4. List BLOCKING / NICE_TO_HAVE / USER_VERIFIES counts.
5. If a bundled template-capabilities snapshot was used (per Step 5), mention the snapshot date and recommend verifying against the live template before committing.

In both cases:
- Don't ask "anything else?"
- Don't add a closing line.
- The user has what they need.

## What this skill does not do

- **Does not implement code.** Code is `/build-from-spec`'s job.
- **Does not validate criteria by running them.** Verification is `/build-from-spec`.
- **Does not move specs to `IMPLEMENTED/`.** That's `/build-from-spec` after a successful build.
- **Does not invoke `/interview-me`.** If the upstream spec has deep gaps that one round of MVP-relevant `[OPEN]` resolution can't fix, push the user back to re-run the upstream skill (`/detailed-specification` or `/extend-features`).
- **Does not modify the upstream spec.** `[OPEN]` markers in the upstream stay untouched even after MVP-relevant ones are resolved here.
- **Does not propose new integrations in project mode.** Detailed-spec integration decisions are immutable at this stage.

## Frontmatter contract

The frontmatter this skill writes is the contract for `/build-from-spec`:

| Field | Value |
|---|---|
| `spec_type` | `"mvp"` |
| `mode` | `"project"` or `"feature"` |
| `name` | Project or feature name (uppercase snake_case, carried from upstream) |
| `date_started` | The date the MVP spec was written (YYYY-MM-DD) |
| `template` | One of `"kitchen-sink-ts"`, `"kitchen-sink-twotier"`, `"statix"` |
| `status` | `"NOT_YET_IMPLEMENTED"` |
| `parent_spec` | The upstream spec's filename (e.g. `DETAILED_PAGEMARK_20260427.md`) |

`/build-from-spec` will validate this frontmatter shape, the presence of `ACCEPTANCE_CRITERIA`, and that every criterion has a number, tag, pre-conditions, and prose. Criterion verification (whether the criterion can be reduced to a passing assertion) is `/build-from-spec`'s responsibility, not this skill's.

## ACCEPTANCE_CRITERIA contract

A note on the verification model: each template's `CLAUDE.md` currently states that "a criterion is verifiable if it reduces to one of: a command exiting 0, an HTTP request returning a Zod-validated response (or a built page DOM assertion for statix)." This skill writes **prose criteria** instead — the build agent figures out which verification form satisfies each prose criterion at build time. Criteria that genuinely can't be machine-verified are tagged `[USER_VERIFIES]`.

This is a deliberate shift from the templates' current contract. The templates' `CLAUDE.md` AC paragraph should be amended in a separate pass to reflect prose-first AC. Until then, the build agent should treat the prose as the source of truth and the template's verification-form list as the menu of mechanisms to choose from.

## Examples

### Example 1 — project mode, clean cut

User points at `SPECIFICATIONS/NOT_YET_IMPLEMENTED/DETAILED_PAGEMARK_20260427.md` (the PageMark detailed spec — a personal reading-tracker for `kitchen-sink-ts`). Says "MVP this".

Skill reads upstream, confirms project mode, confirms `kitchen-sink-ts` matches the repo, reads template capabilities (auth ✓, DB ✓, OpenRouter ✓, Polar/R2 not needed). No `[OPEN]` markers in the upstream.

Skill proposes cut:
- Keep: capture flow, browse/list, today's review tile, BetterAuth email/password (PROVIDED), OpenRouter summarize (CONFIGURE).
- Defer: weekly digest view, ILIKE search filter, tag multi-select filtering.
- Reason: the loop is "capture → review". Browse/list is needed to navigate; search and digest are auxiliary polish that can land after the loop is proven.

User confirms. Skill writes:
- `MVP_20260427_SPEC.md` with 5 sections of AC: auth (1.x), capture (2.x), browse-list (3.x), today's review (4.x), summarize integration (5.x). Total 17 criteria, 14 BLOCKING, 3 NICE_TO_HAVE.
- `MVP_20260427_DEFERRED.md` with deferred items grouped by source ("deferred from this MVP cycle" vs "carried from upstream Out of scope").

Output: file paths + criterion counts.

### Example 2 — feature mode with integration re-enable

User invokes from a repo that already has `IMPLEMENTED/MVP_20260301_SPEC.md` (PageMark v1 shipped). Wants to add monetization. Points at `DETAILED_PAGEMARK_MONETIZATION_20260427.md` from `/extend-features`.

Skill detects feature mode from frontmatter. Cross-checks against `IMPLEMENTED/`: original MVP marked Polar as REMOVE. The new feature requires Polar.

Skill surfaces: "Original MVP marked Polar as REMOVE. This feature requires it. Plan: re-enable Polar in the build (env vars + product setup + checkout wiring). Confirm?"

User confirms. Skill writes:
- `MVP_PAGEMARK_MONETIZATION_20260427.md` with Integrations section noting Polar re-enable, AC sections for pricing-page (1.x), checkout (2.x), webhook handling (3.x), subscription state (4.x), gated features (5.x).
- `MVP_PAGEMARK_MONETIZATION_20260427_DEFERRED.md` listing deferred items (e.g. enterprise tier, annual billing).

### Example 3 — refusal: total cut

User points at a thin upstream spec and says "actually let's defer everything for v2".

Skill: refusal in two paragraphs. There's no MVP if the entire spec is deferred. Either pick at least one user flow worth shipping, or revise the upstream spec to scope down at the detailed level instead.

### Example 4 — forced inline `[OPEN]` resolution

User points at a `DETAILED_*` spec containing `[OPEN: Should the verification flow be email-only or also SMS?]` in the auth section.

Skill: identifies this `[OPEN]` as MVP-relevant (auth is in the MVP). Asks the user inline: "The detailed spec has an open question on verification: email-only or also SMS? For the MVP, I recommend email-only — SMS adds Twilio integration which the template doesn't have. Pick one."

User picks email-only. Skill proceeds with the cut. The MVP spec reflects email-only verification; the upstream detailed spec is left untouched (the `[OPEN]` marker stays).
