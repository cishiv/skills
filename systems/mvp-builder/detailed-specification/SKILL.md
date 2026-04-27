---
name: detailed-specification
description: Use this skill whenever the user wants to turn a software project idea into a complete, stack-aware detailed specification — the kind that captures the full vision before any scope-cutting. Trigger on phrases like "detailed spec", "spec out my idea", "turn this into a project spec", "let's specify this", or any time the user provides a brain dump, interview Q&A, or set of notes describing a new software project they want to build. This skill produces a `DETAILED_{PROJECT_NAME}_{YYYYMMDD}.md` artifact for one of the user's pre-existing template repos (kitchen-sink-ts, kitchen-sink-twotier, statix). It is project-mode only — for adding features to existing projects, use `/extend-features` instead.
---

# detailed-specification

This skill takes messy user input describing a new software project and produces a complete, concrete, stack-aware **detailed specification** — the full vision of what the user wants to build, before any scope-cutting.

The output is the input to `/mvp-specification`, which is responsible for cutting scope. If the detailed spec is already MVP-shaped, the next skill has nothing to do. So bias toward completeness and ambition, not minimalism.

## Boundaries

- **Project mode only.** The first detailed spec for a new repo. For adding features to an existing repo, the user should run `/extend-features` instead.
- **One target template.** The user picks one of three: `kitchen-sink-ts`, `kitchen-sink-twotier`, `statix`. The spec is stack-aware from the start.
- **No `ACCEPTANCE_CRITERIA` section.** That's `/mvp-specification`'s responsibility; the bar is different.
- **No silent decisions.** Every fact in the spec must trace to either user input, a follow-up answer, or an explicit `[OPEN: <question>]` marker. The skill never invents content the user didn't approve.

## Workflow

### Step 1 — Determine surface

The skill runs on one of two surfaces. The choice gates output behavior, validation, and follow-up question delivery.

**Claude Code mode** — true if all of:
- The current working directory contains a `SPECIFICATIONS/` directory, OR can walk up from the cwd to find one within 3 levels.
- Filesystem write tools are available.

In this mode, the skill writes the spec directly into the target repo.

**Not Claude Code mode** — anything else. claude.ai chat, mobile, web, sandboxed environments without a target repo.

In this mode, the skill is responsible for evaluating its environment and choosing the best output mechanism from what's available:
- `.md` artifact rendered inline (always available — primary deliverable)
- Obsidian MCP (if connected — silent mirror to staging for async hand-off)
- The user pasting/copying content out of the chat

The split is binary. If "Claude Code mode" conditions don't all hold, the skill is in "Not Claude Code mode" and must adapt.

Don't announce the surface determination. Apply it silently to subsequent steps.

### Step 2 — Gather inputs

**Always ask the user explicitly where their input lives.** Do not silently scan locations. Acceptable input shapes:

- Interview Q&A files (often from `CLAUDE_Q&A/ANSWERED/` if `/interview-me` is in use)
- Raw markdown brain dumps
- Voice notes (transcribed; user pastes the transcript)
- Existing notes referenced by path (Obsidian or filesystem)
- Pasted text directly in the chat
- PDFs, images (e.g. competitor screenshots) — the user uploads or links them
- Any combination of the above

The skill **may** suggest candidate inputs only when all three of the following hold:
- Obsidian MCP is available
- A `CLAUDE_Q&A/ANSWERED/` directory exists
- The user has `/interview-me` installed

In that case, propose recently-modified Q&A files (within the last 7 days) as candidate inputs. The user confirms or specifies their own.

If none of the conditions hold, simply ask: "Where's the input? Paste it, link a file, or describe what you want me to read."

#### Refusal: input has no buildable idea

A brain dump qualifies as "buildable" if it contains at minimum:
- One user flow (even one word strung together with arrows is okay)
- A stated problem or intention

If the input falls below this bar — pure fragments, no goal, no user — refuse to proceed.

**Refusal format: two paragraphs maximum.** First paragraph: state what's missing concretely (no user flow, no problem statement, etc.). Second paragraph: suggest running `/interview-me` first and name 2-3 specific things to develop during the interview. Don't lecture. Don't pad. The user has lower-friction surfaces to work on the idea.

Do **not** try to write a spec from incoherent input. The downstream pipeline depends on a real spec.

#### Refusal: input contradicts itself

If the input says "should be free" in one place and "premium tier" in another, surface the contradiction explicitly as a follow-up question. Don't silently pick one path. The contradiction itself is a signal that the user hasn't made the decision yet — help them through it rather than committing.

### Step 3 — Confirm project name and target template

**Project name.** Read the input. Propose an uppercase snake_case name based on what the project is about (e.g. `RECIPE_TRACKER`, `BUDGET_REVIEWER`). Tell the user: "I'll call this `<NAME>` — change?" Proceed on confirmation.

**Target template.** Ask the user which of the three templates this project should be built on:

- `kitchen-sink-ts` — single SSR deployable. Pick this for full-stack apps that don't need a separate API tier or multiple clients.
- `kitchen-sink-twotier` — separate Hono backend + React client. Pick this when the API will serve multiple clients, or when there's significant server-side processing that warrants separation.
- `statix` — static markdown site. Pick this for documentation, blogs, or any read-only content site.

If the user names a template that's clearly a mismatch for the input (e.g. real-time multiplayer in `statix`), flag the mismatch and ask the user to confirm or rephrase. Do not silently override.

### Step 4 — Acquire the spec template

The skill writes specs that conform to the target repo's `SPECIFICATIONS/SPEC_TEMPLATE.md`. Do not work from memory — templates evolve.

**Try these sources in order, stopping at the first that works:**

1. **Filesystem.** If the target repo is reachable on disk (typical Claude Code with a working directory inside or near the repo), read `SPECIFICATIONS/SPEC_TEMPLATE.md` directly.
2. **GitHub MCP.** If the GitHub MCP is connected, fetch the file from the `main` branch of the target repo.
3. **`web_fetch`.** If `web_fetch` is available and the repo is public, fetch the raw URL: `https://raw.githubusercontent.com/cishiv/<template>/main/SPECIFICATIONS/SPEC_TEMPLATE.md`.
4. **Bundled snapshot.** Read the corresponding file in the skill's own `references/template-contracts/` directory: `kitchen-sink-ts.md`, `kitchen-sink-twotier.md`, or `statix.md`. Each file includes the literal template inside a code fence and a snapshot date in the header.

If steps 1–3 fail and the bundled snapshot is used, **explicitly tell the user**:
- Which snapshot date is in use.
- That the snapshot may be stale.
- A recommendation to verify the produced spec against the live template before committing.

If even step 4 is unavailable (the bundled directory is missing — should not happen but possible if the skill is partially installed), ask the user to paste the live template contents into the chat. This is the last resort.

Don't proceed without a template source. The output structure must match the repo's contract.

### Step 5 — Gap analysis

Compare the user's input against the spec template's required sections:

- Problem statement
- User flows (need at least one happy path end-to-end)
- Data model (or "Content model changes" for `statix`) — entities should be named and typed; tail concerns are okay to defer
- Architecture hints
- Integrations (KEEP / REMOVE / NOT_PRESENT for each scaffold integration; omit this section for `statix`)
- Out of scope (optional but useful)

Identify each gap. Identify each contradiction. Identify each statement that's ambiguous enough to require a decision.

#### Gaps vs product design

Some gap questions will inevitably touch product design — for example, "how does verification work?" when the input mentions verified users. This is acceptable: when the product structure is unstated, the spec can't be complete without it.

The boundary the skill respects: **ask about decisions the user could make in one or two sentences, not problems they need a designer to solve in a workshop.** "Should verification be manual review or auto-via linked GitHub?" is a gap question — the user picks one. "How should we design the onboarding experience to maximize signup conversion?" is a product question — out of scope for this skill, push back to the user.

If a gap can only be answered with a multi-paragraph design exploration, surface it as a meta-issue: "this part of the product needs design work before it can be specified," and offer to bracket it as `[OPEN: ...]` or recommend the user do a separate design pass first.

### Step 6 — Follow-up questions

Default mode is **single batch**: one round of questions, then write the spec. The user can opt into iterative mode (write draft → surface weak spots → ask → revise) by saying so explicitly. If the user hasn't said anything, default to single batch.

**Question delivery — user choice per invocation.** Ask the user where they want to answer:
- Inline in the chat (faster, lower-friction in claude.ai chat)
- Written to an Obsidian Q&A file at `CLAUDE_Q&A/UNANSWERED/DETAILED_SPEC_{PROJECT_NAME}_{YYYYMMDD}.md` (better in Claude Code, where chat is more transactional)

Propose a default based on environment, but let the user override. Don't ask this question if the user has already specified.

**Hard limit: 20 questions per batch.** If gaps exceed 20, ask the most-blocking 20, write a partial spec with `[OPEN: <question>]` markers for the rest, and explicitly tell the user the spec is partial.

#### Handling weak answers

If the user says "idk" or skips a question:
- Write the spec with an explicit `[OPEN: <question>]` marker in the relevant section.
- Do not mark the spec complete (this is signaled to downstream skills via the open markers).

If the user wants to genuinely **ignore** a question rather than answer it, require a one-sentence rationale for why it doesn't matter. The rationale goes into the spec as a comment so future readers understand why the gap was accepted. Without the rationale, treat it as `[OPEN]`.

### Step 7 — Pre-write validation

Before writing, validate:

1. The target repo exists and has a `SPECIFICATIONS/NOT_YET_IMPLEMENTED/` directory.
2. The template in the frontmatter matches the repo (the repo's own `CLAUDE.md` or `README.md` typically declares which template it is).
3. There's no filename collision with an existing `DETAILED_{PROJECT_NAME}_{YYYYMMDD}.md` in `NOT_YET_IMPLEMENTED/`.

**Validation by surface:**

- **Claude Code with filesystem access to the repo:** validate programmatically.
- **Claude Code without the repo cloned, with GitHub MCP available:** ask the user to link the repo, then validate.
- **No filesystem and no GitHub MCP:** ask the user to validate manually. Provide specific instructions for each check, e.g.: "Run `ls SPECIFICATIONS/NOT_YET_IMPLEMENTED/` in the repo and confirm there's no file named `DETAILED_<PROJECT_NAME>_<YYYYMMDD>.md`."
- **claude.ai chat (output going to Obsidian staging):** check the Obsidian staging path for collision instead.

#### Refusal: collision on same project, same day

If a `DETAILED_{PROJECT_NAME}_{YYYYMMDD}.md` already exists, refuse and surface the collision to the user. Two options for them:
- Explicitly say "overwrite" → proceed and overwrite.
- Pick a different name or come back tomorrow.

Do **not** silently bump to `_v2` or overwrite. The user must explicitly approve destruction.

### Step 8 — Write the spec

**Filename:** `DETAILED_{PROJECT_NAME}_{YYYYMMDD}.md`

**Output location, by surface:**

- **Claude Code mode:** write to `<repo>/SPECIFICATIONS/NOT_YET_IMPLEMENTED/<filename>` directly. The file in the repo is the canonical deliverable.
- **Not Claude Code mode:** the canonical deliverable is an `.md` artifact rendered inline in the response. If Obsidian MCP is connected, ALSO silently write to `AI/Skill Systems/MVP_SYSTEM/STAGING/<filename>` as a backup for async hand-off — mention the staging path in passing, but don't make it the primary surface.

**Frontmatter** (exactly this shape):

```yaml
---
spec_type: "detailed"
mode: "project"
name: "<PROJECT_NAME>"
date_started: "<YYYY-MM-DD>"
template: "<chosen template>"
status: "NOT_YET_IMPLEMENTED"
parent_spec: ""
---
```

`parent_spec` is empty — this is a project-mode detailed spec, which has no upstream spec.

**Drop the frontmatter usage comment block** that the bundled template snapshot contains (the `<!-- Frontmatter usage: ... -->` block immediately after the YAML). That block is template guidance for spec authors, not part of the spec content. The produced spec should jump straight from the closing `---` of the frontmatter to the `# <Project name>` heading.

**Sections** (per the target repo's `SPEC_TEMPLATE.md`):

1. Problem statement
2. User flows
3. Data model changes (or "Content model changes" for `statix`)
4. Architecture hints
5. Integrations (omit this section entirely for `statix`)
6. Out of scope

**Omit the `ACCEPTANCE_CRITERIA` section entirely.** Don't include an empty placeholder. The MVP skill's bar is different and the empty section creates noise.

### Step 9 — Self-check

After writing, verify:

- All required sections are present (per the template — remember `statix` differences).
- No template placeholder text survives in the output (`<short feature name>`, `<YYYY-MM-DD>`, `<Why this feature exists...>`, etc.).
- Frontmatter is valid YAML.
- All `[OPEN: ...]` markers correspond to actual unanswered questions, not accidental leftovers.
- No accidental ACCEPTANCE_CRITERIA section.

If any check fails, fix it. If a fix isn't possible (e.g. template mismatch), surface the failure to the user.

### Step 10 — Output to user

Output behavior depends on surface (per Step 1).

**Claude Code mode:**
1. Confirm the file path where the spec was written (e.g. `SPECIFICATIONS/NOT_YET_IMPLEMENTED/DETAILED_PAGEMARK_20260427.md`).
2. List any `[OPEN: <question>]` markers and any rationales for ignored questions.
3. Don't dump the full spec content into chat — the file is the deliverable.

**Not Claude Code mode:**
1. The `.md` artifact rendered inline IS the deliverable. Don't repeat the content as plain chat text afterward.
2. If Obsidian staging was used, mention the staging path briefly so the user knows where the backup lives.
3. List any `[OPEN: <question>]` markers and any rationales for ignored questions.
4. If a bundled snapshot was used (per Step 4), mention the snapshot date and recommend verifying against the live template before committing.

In both cases:
- Don't ask "anything else?"
- Don't add a closing line.
- The user has what they need.

## What this skill does not do

- **Does not invoke `/interview-me` automatically.** If input is too thin, suggest the user run it. After the suggestion, control returns to the user — they re-invoke this skill when they have better input.
- **Does not write the MVP spec.** That's `/mvp-specification`'s job. This skill produces the detailed (full vision) spec.
- **Does not handle feature additions.** Feature mode goes through `/extend-features`, which is a sibling entry-point skill.
- **Does not deploy, build, or modify code.** Spec-only.
- **Does not move specs between `NOT_YET_IMPLEMENTED/` and `IMPLEMENTED/`.** That's `/build-mvp`'s responsibility after a successful build.

## Frontmatter contract

The frontmatter this skill writes is the contract for downstream skills:

| Field | Value |
|---|---|
| `spec_type` | `"detailed"` |
| `mode` | `"project"` |
| `name` | The project name (uppercase snake_case) |
| `date_started` | The date the spec was written (YYYY-MM-DD) |
| `template` | One of `"kitchen-sink-ts"`, `"kitchen-sink-twotier"`, `"statix"` |
| `status` | `"NOT_YET_IMPLEMENTED"` |
| `parent_spec` | `""` (always empty for project-mode detailed specs) |

`/mvp-specification` validates these fields before consuming the spec. An empty `parent_spec` is the unique signature of a project-mode detailed spec; if the field is non-empty, this skill produced it incorrectly.

## Examples

### Example 1 — clean input, single batch

User pastes a 500-word brain dump describing a recipe-tracking web app. Mentions login, recipe CRUD, sharing recipes with friends. Picks `kitchen-sink-ts`.

Skill identifies four gaps: no data model details on the "sharing" feature, no decision on whether recipes are public/private by default, no mention of search/filter, no integrations decision. Asks 4 questions in one batch. User answers in chat. Skill writes the spec to `SPECIFICATIONS/NOT_YET_IMPLEMENTED/DETAILED_RECIPE_TRACKER_20260427.md`. Output is the full spec inline + the path. No `[OPEN]` markers.

### Example 2 — thin input, refusal

User pastes "I want to make something for tracking habits maybe with friends." Three sentences total. No user flow, no clear problem statement.

Skill refuses. Tells the user: "This is below the buildable bar — there's no user flow and the problem isn't stated. Suggest running `/interview-me` first to develop the idea. Come back when you have at least one user flow described and a clear problem statement."

### Example 3 — contradictions

User's brain dump says the app is free in one paragraph and mentions a "Pro tier with $5/mo" in another. Skill picks this up in gap analysis.

Question batch includes: "The input contradicts itself on monetization — should the v1 be free, freemium with a paid tier, or paid-only?" User picks freemium. Spec reflects the resolution; the contradiction doesn't appear in the output.

### Example 4 — partial spec with `[OPEN]` markers

User answers 18 of 20 follow-up questions. Two questions get "idk" with no rationale.

Skill writes the spec, but the relevant sections include `[OPEN: <question>]` markers verbatim. The user is told the spec is incomplete and which questions remain. The downstream MVP skill will see the markers and refuse to consume the spec until they're resolved.
