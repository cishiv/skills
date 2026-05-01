---
name: update-next
description: Use this skill to log "next steps" from the current chat session into the Obsidian file `AI/NEXT.md`. Trigger on `/update-next`, "log next steps", "update what's next", "add to NEXT", or any time the user asks to capture forward-looking work for this project before ending the session. Each project has its own H3 section in `AI/NEXT.md` with a nested checklist of next items (`- [ ] item`). Items can be for the human or the agent — the skill doesn't distinguish, just captures the action. The skill reads the file, finds the matching H3 (or creates one), then adds new items / marks completed ones / edits stale ones based on what the session has surfaced.
---

# update-next

Capture next-step actions from the current chat session into `AI/NEXT.md` in Obsidian. Each project gets one H3 section with a nested checklist; items are forward-looking work that's either pending the human or pending the agent.

The intent is simple: when ending a session, run this skill so the next session (or the next terminal context) opens with a clear picture of what was left undone.

## File shape

```markdown
### {project name 1}
- [ ] Item one
- [ ] Item two
  - [ ] Nested sub-item
- [x] Item three (already done)

### {project name 2}
- [ ] Different project's item
```

Sections are H3. Items are markdown checkboxes (`- [ ] ` or `- [x] `). Nesting is two-space indented. The file lives at `AI/NEXT.md` (vault-relative).

## Workflow

### Step 1 — Identify the project section

Determine which H3 section to update. In order:

1. If the user named the project ("/update-next for skills repo", "log this to mvp-builder"), use that.
2. Otherwise, derive from the current working directory's basename (`basename $(pwd)`). For example `/Users/shiv/personal/skills` → `skills`; `/Users/shiv/personal/templates/kitchen-sink-ts` → `kitchen-sink-ts`.
3. If the cwd basename is generic or ambiguous (e.g. `tmp`, `home`), ask the user.

Don't guess silently. If the derived name is plausible, propose it and proceed; if it isn't, ask.

### Step 2 — Synthesize the next-step items

Look at what the current chat session has been doing. Identify forward-looking actions that are NOT yet done — either because they're queued for the human (review a PR, configure something in a dashboard, answer a Q&A) or queued for the agent (resume a build, scope a skill, refresh a snapshot).

Each item should be:

- **One short line.** No paragraphs. Imperative phrasing ("Run the dogfood", "Drag Q&A files to ANSWERED").
- **Self-contained.** A future reader (human or agent) can act on it without re-reading the chat.
- **Actionable.** Not "think about X" — name the action.
- **Audience-agnostic.** Don't tag "for human" / "for agent"; the action speaks for itself.

If the session produced many candidate items, surface them in chat first and let the user prune before writing. Don't dump 15 items into NEXT.md unprompted.

### Step 3 — Read `AI/NEXT.md`

Use `obsidian_get_file_contents` to read the current file.

If the file doesn't exist (404 or empty), proceed to Step 5 with an empty starting state.

### Step 4 — Match the H3 section

Look for an H3 that matches the project name from Step 1. Match is case-insensitive on the heading text (exact word match, ignoring leading/trailing whitespace). If multiple H3s match, ask the user which one.

If no match exists, the section will be created (Step 5).

### Step 5 — Reconcile and write

Compare the synthesized items against what's already under the section (if it exists):

- **Already-checked items** (`- [x]`) → leave alone.
- **Unchecked items still pending** (`- [ ]`) → keep as-is. If the session resolved one, mark it `- [x]` instead.
- **New items** → add as new `- [ ]` lines under the section.
- **Stale items** (the session made one obsolete by going in a different direction) → mark as `- [x]` with a strikethrough comment, or remove. Ask the user if it's not obvious.

**If the section exists:**

Use `obsidian_patch_content` with `target_type: "heading"`, `target: "<heading text>"`, `operation: "append"`, and the new/updated checklist content. Patch only the items that changed; leave the rest of the section untouched.

For more complex updates (marking existing items done, editing stale items), read the section, modify the checklist in memory, and use `obsidian_patch_content` with `operation: "replace"` against the heading to overwrite the section body.

**If the section doesn't exist:**

Use `obsidian_append_content` to add a new H3 + checklist at the end of the file:

```markdown

### {project name}
- [ ] Item one
- [ ] Item two
```

Leading blank line so the new section is separated from prior content.

### Step 6 — Confirm to the user

In chat, in 1-2 lines:

- The section name updated/created.
- How many items added, updated, or marked done.

Don't dump the full file content.

## Edge cases

- **Empty session.** If there are no forward-looking items to capture (the session ended cleanly with nothing pending), say so and don't write anything. The user invoked the skill expecting something to log; if there isn't, tell them.
- **Vague items.** If the session's work didn't produce concrete next steps but only vague directions ("revisit later"), surface those vaguely back to the user and ask if they want to log them as-is or skip.
- **Multiple project contexts in one session.** If the session touched multiple projects (e.g. you worked on the skills repo AND amended the templates), ask the user which section(s) to update — or do multiple sections in one go if the user confirms.
- **`AI/NEXT.md` doesn't exist yet.** Create it with the first section. Don't over-engineer initial scaffolding — a single H3 section is fine.
- **Items that are partially done.** Leave them as `- [ ]`. Add a sub-item `  - [x] <progress note>` if it's useful to record progress, but don't mark the parent done until it actually is.

## What this skill does not do

- **Does not invent next steps.** Only logs what the session genuinely surfaced. If the user wants more aspirational planning, that belongs in a longer document (META_SCOPE, project specs), not in NEXT.md.
- **Does not delete sections.** Only appends to or modifies the targeted section. Other sections in `AI/NEXT.md` are left untouched.
- **Does not tag items by audience.** Human vs agent doesn't matter at log time; the action is the action.
- **Does not commit anything.** This is purely an Obsidian write. Git state is unaffected.
- **Does not promise to be a task tracker.** This is an interim shape until Tasker takes over. Don't accrete features.
