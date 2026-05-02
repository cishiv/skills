---
name: linkedin-post-ideas
description: Use this skill to surface a week's worth of LinkedIn post ideas anchored to the user's brand pillars and recent work. Trigger on `/linkedin-post-ideas`, "what should I post this week", "give me LinkedIn ideas", "weekly post ideas". Reads recent activity from GitHub (recent commits — via GitHub MCP first, `gh` CLI fallback if on Claude Code), Obsidian (vault-wide recent changes over the last 7 days), and prior LinkedIn posts (`Writing/LinkedIn/Drafts/` + `Writing/LinkedIn/Published/`, distinguished). Reads brand artifacts from `AI/PersonalBrand/` (pillars REQUIRED — refuses without them) and optionally a tone reference from `Writing/ToneReferences/` (used for hook drafting). Produces 14-21 ideas spread across 7 days from invocation date (2-3 per day default; can be 1 if material is thin), each ranked within its day with rationale. Each idea is a checkbox the user can flip when drafted, plus a copy-paste-friendly invocation for `/linkedin-post-synthesis`. Mirrored to chat artifact and Obsidian (`Writing/LinkedIn/Ideas/{week-of-YYYY-MM-DD}.md`). Works on any surface that has the Obsidian MCP. Strict pillar mapping — ideas that don't fit a pillar are dropped, not flagged.
---

# linkedin-post-ideas

Surface a week's worth of LinkedIn post ideas anchored to your brand pillars and grounded in your recent work. The intent is to remove "what do I post about?" friction at the start of a week — you get a backlog of 14-21 candidate ideas, pick 7 to publish, and each idea is a copy-paste away from being drafted via `/linkedin-post-synthesis`.

The skill is opinionated about anchoring: **brand pillars are required**. Ideas that don't map to a pillar get dropped (not flagged). The reasoning: if an idea doesn't fit your brand, you're going off-brand — and that's a decision worth making manually, not nudged toward by an ideas generator.

## Boundaries

- **Any surface with the Obsidian MCP.** Bash + filesystem (Claude Code) unlocks `gh` CLI fallback; otherwise GitHub MCP is the only GitHub source.
- **Brand pillars REQUIRED.** No `AI/PersonalBrand/pillars.md` (or `index.md` referencing pillars) → refuse. Push the user to `/personal-brand-coach` first.
- **Tone reference OPTIONAL.** When present (from `Writing/ToneReferences/_registry.md`, ideally one with `intended_use: linkedin-posts`), the skill uses it for hook drafting. When absent, hooks stay neutral.
- **Strict pillar mapping.** Each generated idea must map to exactly one pillar. Ideas that don't fit any pillar are dropped during synthesis, not surfaced with an off-pillar flag.
- **7 days from invocation, not Mon-Sun.** Day 1 is the invocation date; Day 7 is invocation + 6 days. Affects the file name and the per-day grouping.
- **Per-day cadence.** Ideas are time-anchored across the 7 days. Each day gets 2-3 ideas (or 1 if material is thin). The user picks one per day to publish (~7 of the 14-21).
- **Strict avoidance of prior topics.** Topics covered in `Writing/LinkedIn/Drafts/` or `Writing/LinkedIn/Published/` within the time window are not regenerated, UNLESS the new idea is a logical continuation (e.g. a deeper take on something already touched briefly).
- **Source warning, not refusal, on partial sources.** GitHub fails → proceed with Obsidian + prior posts + a warning. Obsidian recent-changes empty → proceed with GitHub + prior posts + a warning. All sources empty → ask the user for input rather than refuse outright.

## Workflow

### Step 1 — Detect surface and tool availability

Detect surface:

- Bash + filesystem available → Claude Code.
- Otherwise → claude.ai.

Tool inventory:

- **GitHub MCP** — call a no-op (e.g. list authorized user) to confirm responsiveness.
- **`gh` CLI** (Claude Code only) — `gh auth status` to confirm available + authenticated.
- **Obsidian MCP** — required. Refuse outright if absent.

### Step 2 — Pre-flight: brand pillars required

Read `AI/PersonalBrand/index.md` (preferred) or `AI/PersonalBrand/pillars.md` directly.

#### Refusal: no brand pillars

If neither exists, refuse in two paragraphs. First: state that ideas need to anchor to brand pillars and none were found. Second: tell the user to run `/personal-brand-coach` first to bootstrap pillars (and other brand artifacts), then re-invoke this skill.

Capture the pillar names + claims for use in synthesis (Step 5).

### Step 3 — Pre-flight: tone reference (optional)

Read `Writing/ToneReferences/_registry.md`. Pick the entry with `intended_use: linkedin-posts` (preferred); fall back to the most recent if none match.

If no tone references exist, proceed without — hooks will be drafted in a neutral tone with a note in the output that they'd be improved with `/generate-tone-reference`.

### Step 4 — Gather source material (last 7 days)

#### GitHub recent commits

Try in order:

1. **GitHub MCP** — fetch recent commits across the user's repos. If MCP returns nothing or errors, retry once.
2. **`gh` CLI** (Claude Code only) — `gh api /users/{user}/events --paginate` filtered to push events within 7 days.
3. **Skip GitHub** — note in the output that GitHub data is missing.

For each commit, capture: repo, message, date. Group by repo so the synthesis can reason about "what was being worked on in repo X."

#### Obsidian recent changes

Use `obsidian_get_recent_changes` to get vault-wide changes within the last 7 days.

For each change, capture: path, modification date, brief content summary (first 200 chars of the changed file or just the title if change is small). Filter out:

- Files in `Writing/LinkedIn/` (those are handled separately as prior posts).
- Files in `CLAUDE_Q&A/` (interview Q&As are noisy for this purpose).
- Files in `AI/NEXT.md` (meta).

#### Prior LinkedIn posts

Read both:

- `Writing/LinkedIn/Drafts/` — drafts within the last 7 days.
- `Writing/LinkedIn/Published/` — published posts within the last 7 days (if the path exists; otherwise skip).

Distinguish drafts vs published in the source set. Ideas can build on either, but the avoidance rule (Step 5) treats them differently: published topics are off-limits for repetition; draft topics MAY be revisited if the user hasn't shipped them yet (but flag as "you have an unpublished draft on this topic — iterate or fresh?").

#### All sources empty

If GitHub returns nothing AND Obsidian recent-changes is empty AND there are no recent LinkedIn posts to build on, **don't refuse outright**. Ask the user: "Quiet week across all sources. Want to: (a) provide some inputs (broad topics, what's been on your mind), (b) generate ideas from your pillars in pure synthesis mode (no recent-work anchoring), or (c) abort?"

### Step 5 — Synthesize candidate ideas

From the source material, generate candidate ideas. For each candidate:

- **Topic** — one phrase capturing the idea.
- **Angle** — the spin / point of view.
- **Hook draft** — one-line opener (uses the tone reference's structural patterns if available; neutral tone otherwise).
- **Pillar tag** — which pillar this idea serves.
- **Source citation** — what surfaced this idea ("Inspired by your `feat(auth)` commits in `cishiv/probe` this week" / "Builds on your published post on EARS criteria from 3 days ago").

#### Filtering

- **Pillar filter** — drop candidates that don't map cleanly to any pillar. No off-pillar flag — they're just dropped.
- **Avoidance filter** — drop candidates whose topic was covered in `Writing/LinkedIn/Published/` within 7 days, unless the candidate is a logical continuation (e.g. "deeper take on X" / "follow-up to X with new evidence").
- **Deduplication** — collapse candidates that are essentially the same with slight angle variations. Pick the strongest variation.

#### Pillar-vs-recent-work disconnect

If recent work clusters around topics that don't match the brand pillars (e.g. you've been deep in ML for two weeks, but pillars say "people management"):

- Generate ideas from the recent work anyway (it's where the signal is).
- Surface the disconnect in the output: "Note: your recent work skews toward <topic cluster>, but your pillars are <pillars>. The ideas below reflect the work, not the brand. Worth reviewing brand-coach if this isn't a one-off."
- Let the user decide whether to publish the off-pillar-but-real ideas, revise the pillars, or both.

### Step 6 — Distribute across 7 days

Spread the surviving ideas across 7 days from the invocation date.

- **Day 1 = invocation date.** Day 7 = invocation + 6.
- **Default 2-3 ideas per day.** Total target: 14-21 ideas.
- **Days can have 1 idea** if material is thin for that day's slot.
- **Rank within day.** For each day, ideas are ranked #1 (strongest), #2, #3 by signal strength + pillar alignment. Each idea includes a one-line "Why ranked here" rationale.

Distribution heuristic: pillar coverage is **agnostic** — don't force each pillar to get a day. Just put the strongest ideas in the strongest slots. If one pillar dominates the recent work, it dominates the week; that's a feature, not a bug.

### Step 7 — Write the output

#### Filename

`Writing/LinkedIn/Ideas/{week-of-YYYY-MM-DD}.md` where `YYYY-MM-DD` is the invocation date (Day 1).

#### Same-week collision

If the file already exists for this week, **append a new session block** at the end. Don't refuse, don't replace. Each session block is dated and the user can compare what surfaced across sessions.

#### File structure

```markdown
# LinkedIn post ideas — week of {YYYY-MM-DD} ({Day-of-week})

## Session 1 — {YYYY-MM-DD HH:MM}

**Brand source:** AI/PersonalBrand/ (pillars: <list>)
**Tone source:** <Writing/ToneReferences/path or "(none — hooks are neutral)">
**Sources scanned:** GitHub <N commits across M repos> | Obsidian <N recent changes> | LinkedIn drafts <N> | Published <N>
**Time window:** Last 7 days (<from-date> to <to-date>)
**Disconnect note:** <if applicable, the pillar-vs-recent-work disconnect summary>

---

### Day 1 — {Mon 5 May 2026}

- [ ] **1.1: {topic phrase}** [pillar: {pillar name}] — ranked #1
  - **Angle:** {one-sentence spin}
  - **Hook draft:** {one-line opener}
  - **Source signal:** {citation}
  - **Why ranked here:** {one-line rationale}

  *To draft:* `/linkedin-post-synthesis Topic: {topic}. Theme: {pillar + angle in one phrase}. Brain dump: {2-3 sentences expanding the angle and source signal}.`

- [ ] **1.2: {topic phrase}** [pillar: {pillar name}] — ranked #2
  - **Angle:** ...
  - **Hook draft:** ...
  - **Source signal:** ...
  - **Why ranked here:** ...

  *To draft:* `/linkedin-post-synthesis ...`

[continue for 1.3 if applicable]

---

### Day 2 — {Tue 6 May 2026}

[same shape — 2.1, 2.2, [2.3]]

---

[Days 3-7]
```

The checkbox at top-level lets the user mark which ideas got drafted. Post-draft tracking is otherwise out of scope — neither this skill nor `/linkedin-post-synthesis` automatically updates the checkbox.

#### Chat output

Render the same content as a `.md` artifact in chat. Plus a 2-line summary:

```
{N} ideas across 7 days written to Writing/LinkedIn/Ideas/{week-of-YYYY-MM-DD}.md (session 1).
{Disconnect note if applicable, in one line.}
```

## What this skill does not do

- **Does not draft posts.** Surfaces ideas with hook drafts; full drafting is `/linkedin-post-synthesis`'s job.
- **Does not generate off-pillar ideas.** They get dropped, not flagged. The user's brand boundaries are respected.
- **Does not auto-fall-through to other skills.** If the user wants to draft an idea, they invoke `/linkedin-post-synthesis` themselves (with the copy-paste-friendly invocation in the ideas file).
- **Does not track which ideas got published.** Checkboxes are for the user to flip manually.
- **Does not refuse on missing GitHub or Obsidian sources.** Warns and proceeds with whatever's available. Only refuses on missing brand pillars.
- **Does not commit anything.** Pure Obsidian write + chat output.
- **Does not regenerate covered topics.** Strict avoidance, except for logical continuations.

## Frontmatter contract (output file)

```yaml
---
type: linkedin-post-ideas
week_of: <YYYY-MM-DD>  # Day 1 = invocation date
generated: <YYYY-MM-DD HH:MM>  # session 1 timestamp
brand_source: AI/PersonalBrand/
tone_source: <path or "(none)">
total_ideas: <N>
pillars_covered: <list>
sessions: <count>  # incremented on append
---
```

## Examples

### Example 1 — full anchoring, healthy week

User: "/linkedin-post-ideas"

Skill: detects Claude Code, GitHub MCP available. Reads `AI/PersonalBrand/index.md` (3 pillars). Reads `Writing/ToneReferences/_registry.md` → finds `linkedin-personal.md` (intended_use: linkedin-posts).

Sources:
- GitHub: 14 commits across 3 repos this week.
- Obsidian: 22 recent changes (filters down to 8 substantive after excluding `LinkedIn/`, `CLAUDE_Q&A/`, `AI/NEXT.md`).
- LinkedIn drafts: 2 from this week.
- Published: 1 from this week.

Synthesizes 24 candidate ideas → drops 6 off-pillar → drops 2 covered by published post → dedups 1 → 15 surviving ideas. Distributes 2-3 per day across 7 days from invocation date.

Writes `Writing/LinkedIn/Ideas/week-of-2026-05-02.md` with 15 ideas, ranked within each day, each with hook draft + copy-paste-friendly synthesis invocation.

Chat: "15 ideas across 7 days written to Writing/LinkedIn/Ideas/week-of-2026-05-02.md (session 1)."

### Example 2 — refusal: no brand pillars

User: "/linkedin-post-ideas"

Skill: reads `AI/PersonalBrand/` — directory doesn't exist. Refuses:

> No brand pillars found in `AI/PersonalBrand/`. This skill anchors ideas to your pillars (so they're not generic) and drops anything that doesn't fit — without pillars, there's nothing to anchor against.
>
> Run `/personal-brand-coach` first to bootstrap your brand artifacts (pillars, audience, positioning, anti-positioning, light voice). The first session is ~1hr and produces a working brand reference. Then re-invoke this skill.

### Example 3 — pillar-vs-work disconnect

User: "/linkedin-post-ideas" in a week where their commits are all infra / DevOps but pillars are "product strategy + agile leadership."

Skill: synthesizes candidates from the infra work, finds none map cleanly to the pillars. Per the disconnect rule, generates from the work anyway and surfaces:

> **Disconnect note:** Your recent work this week (14 commits across `infra-tooling`, `monitoring-dashboard`) skews technical / DevOps. Pillars are product strategy + agile leadership. The ideas below reflect the work, not the brand. Worth reviewing `/personal-brand-coach` if this isn't a one-off.

Then lists ideas like "What I learned migrating to ECS Fargate" — clearly off-brand-pillar, but real. User can publish the off-pillar ideas, revise pillars, or skip the week.

### Example 4 — same-week append

User invokes `/linkedin-post-ideas` on Tuesday. File `week-of-2026-05-02.md` already exists from Monday's invocation.

Skill: appends `## Session 2 — 2026-05-03 09:14` to the existing file. New ideas may overlap or differ from session 1 (the source window has shifted by a day, and new commits / changes may have appeared).

Chat: "Appended session 2: 12 new ideas across 7 days to Writing/LinkedIn/Ideas/week-of-2026-05-02.md (now 2 sessions)."

### Example 5 — quiet week

User: "/linkedin-post-ideas" during a holiday week with zero commits, no Obsidian changes, no recent posts.

Skill: doesn't refuse. Asks: "Quiet week across all sources (GitHub: 0 commits, Obsidian: 0 recent changes, LinkedIn: 0 recent posts). Pick: (a) provide some inputs (broad topics, what's been on your mind), (b) generate from pillars only (no recent-work anchoring — ideas will be more general), (c) abort."

User picks (b). Skill generates 7-14 ideas (lower count given no signal density) anchored purely to pillars. Output flags "pillars-only mode, no recent-work anchoring."
