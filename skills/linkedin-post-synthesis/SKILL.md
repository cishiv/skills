---
name: linkedin-post-synthesis
description: Use this skill to draft LinkedIn posts in the user's voice. Trigger on `/linkedin-post-synthesis`, "draft a LinkedIn post about X", "write a post on Y", "let's write something for LinkedIn". Takes a topic + theme + brain dump, plus optional supporting context (paste, URLs, Obsidian paths, PDFs, screenshots, voice transcripts), and produces 3 post variants with rationale. Always combines the brand artifacts from `/personal-brand-coach` (`AI/PersonalBrand/`) AND a tone reference from `/generate-tone-reference` (`Writing/ToneReferences/*.md`) — both are required. There is NO bundled default fallback. If neither tone source exists, the skill refuses and pushes the user to bootstrap one. Drafts are mirrored to chat (artifact) and Obsidian (`Writing/LinkedIn/Drafts/{topic-slug}_{YYYYMMDD}.md`). Iterative refinement happens within the same invocation; the skill remembers prior drafts in-session. Works on any surface.
---

# linkedin-post-synthesis

Draft LinkedIn posts in your voice using your brand pillars and tone reference. Three variants per invocation, each with a brief rationale showing why it works for the brand and which pillar it serves. No template-y AI output — the variants are governed by the tone reference, not by generic LinkedIn-post heuristics.

The skill assumes you've done the prep: `/personal-brand-coach` for brand pillars + audience + positioning, and `/generate-tone-reference` for at least one tone reference (typically `Writing/ToneReferences/linkedin-personal.md` or similar). Both are required — the skill refuses without them rather than fall back to a generic default.

## Boundaries

- **Any surface with the Obsidian MCP.** Reads brand artifacts and tone references from Obsidian; mirrors drafts there too.
- **Brand artifacts AND tone reference are both required.** No bundled fallback. If neither exists → refuse and tell the user to bootstrap via `/personal-brand-coach` or `/generate-tone-reference`.
- **Hybrid resolution always.** When both brand artifacts and a tone reference exist, the skill combines them — pillars / audience / positioning come from the brand, voice / structural patterns / vocabulary come from the tone reference. Skill does not "fall through" from one to the other; partial overlap is normal and intended.
- **Auto-resolves the tone source.** Skill picks the brand artifacts and the most appropriate tone reference (matching `intended_use: linkedin-posts` from the registry) silently. Surfaces what it picked in the output but doesn't ask up front.
- **3 variants with rationale.** Always. Not 1, not 5. Each variant shows which pillar it speaks to and why the structural choice works.
- **Tone reference governs everything formal.** Length, hooks, CTA presence, emoji use, line breaks — all per the tone reference, not a generic "LinkedIn-native" template. The ~1300 char native length is an upper bound, not a target.
- **Same invocation iteration.** User refines ("more direct", "drop the emoji", "shorter") within the same chat session. Skill remembers prior drafts so the user doesn't have to re-paste them.
- **Off-brand requests get surfaced.** If the user asks for a post that contradicts the pillars, the skill surfaces the contradiction and asks the user to confirm the deviation before drafting.

## Workflow

### Step 1 — Pre-flight: tone resolution

Read in this order:

1. **Brand artifacts** — `AI/PersonalBrand/index.md` (preferred) or canonical files (`pillars.md`, `audience.md`, `positioning.md`, `voice-light.md`).
2. **Tone references registry** — `Writing/ToneReferences/_registry.md`.
3. **Best-match tone reference** — from the registry, pick the entry with `intended_use: linkedin-posts`. If none with that intended_use, pick the most recent. If multiple ties, pick the one with the highest `confidence`.

Refusal cases:

- **No brand artifacts AND no tone references** → refuse. Two-paragraph format. First: state that this skill requires either `AI/PersonalBrand/` artifacts or at least one `Writing/ToneReferences/*.md`. Second: tell the user to invoke `/personal-brand-coach` first (preferred — produces both pillar context AND a light voice anchor) or `/generate-tone-reference` (sufficient for tone-only; pillar coverage will be missing).
- **Only brand artifacts, no tone references** → proceed but warn: "No tone reference found in `Writing/ToneReferences/`. The drafts will be voice-anchored only by `voice-light.md` from your brand artifacts. Recommend running `/generate-tone-reference` against your prior writing for tighter voice match."
- **Only tone references, no brand artifacts** → proceed but warn: "No brand artifacts found in `AI/PersonalBrand/`. The drafts will use the tone reference's voice but won't be anchored to specific pillars or audience. Recommend running `/personal-brand-coach` to anchor."

### Step 2 — Gather inputs

Required:

- **Topic** — the subject of the post.
- **Theme** — the angle / point of view.
- **Brain dump** — what you actually want to say. Free-form prose. Can be terse or substantial.

Optional supporting context (any combination):

- Pasted text in chat.
- URLs (skill fetches via `web_fetch`).
- Obsidian paths (file or folder).
- PDFs / screenshots / images (uploaded by the user).
- Voice transcripts (pasted as text).

### Step 3 — Audience inference

Skill infers the target audience for this specific post by combining:

- The brand `audience.md` (the canonical audience for the user's writing).
- The topic + brain dump (some topics naturally land for sub-audiences within the canonical set — e.g. an infra-deep post lands for senior engineers within a broader "founders + senior eng" audience).

Surface the inferred audience to the user in the output: "Target audience for this post: <description>. Pulled from `audience.md` + topic shape." If the user wants to override, they say so and the skill re-runs with the override.

### Step 4 — Off-brand detection

Cross-check the topic + theme against `pillars.md` and `anti-positioning.md`:

- **Topic clearly maps to a pillar** → flag the pillar in the rationale. Proceed.
- **Topic doesn't map to any pillar** → flag as `off-pillar`. Don't refuse; surface to the user: "This topic doesn't directly map to your pillars (closest is X but it's a stretch). Proceed?" — only after explicit confirmation.
- **Topic contradicts `anti-positioning.md`** → ask the user explicitly: "This sits in your anti-positioning ('<exact item from anti-positioning>'). Proceed anyway with a deliberate deviation, or re-think the angle?"

### Step 5 — Same-day draft check

Look in `Writing/LinkedIn/Drafts/` for files matching the topic slug + recent date (within last 7 days). If a recent draft exists with similar topic:

- Surface it to the user: "There's a recent draft on a similar topic at `Writing/LinkedIn/Drafts/<file>` (<date>). Iterate on that, or produce a fresh set?"
- Wait for direction. Don't silently produce a new variant set.

### Step 6 — Draft three variants

Produce three variants. Each variant should:

- Be substantively different (different opening pattern, different angle, different structural shape — not three rephrasings of the same post).
- Stay within the tone reference's vocabulary, structural patterns, and do/don't list.
- Honor the length the tone reference implies (or stay within ~1300 chars upper bound if the tone reference is silent on length).
- Use formatting (line breaks, emoji, bullets) per the tone reference's allowance — not generic LinkedIn norms.
- Include or omit a CTA per the tone reference.

#### Hooks

Generate hooks based on the tone reference's `Structural patterns / Openings`. Don't reach for a generic "contrarian take / pattern interrupt / question" library — the tone reference describes how this user opens posts; use that.

If the tone reference is light on opening patterns (low confidence reference), draw on the brand `voice-light.md` for personality cues (warm vs. cool, direct vs. indirect, etc.).

#### Pillar mapping

For each variant, map it to a specific pillar from `pillars.md`. Include in the rationale: "Variant 1 — speaks to pillar: <name>." If a variant doesn't cleanly map (and was approved off-pillar in Step 4), flag in the rationale.

### Step 7 — Output

In chat, render an `.md` artifact containing the three variants in one structured document:

```markdown
# {topic}

**Audience:** <inferred audience>
**Tone source:** <which brand artifacts + which tone reference were used>
**Pillar coverage:** <which pillars are spoken to across the variants>

---

## Variant 1: <short distinguishing label>

**Pillar:** <pillar name>
**Hook pattern:** <which opening pattern this uses, per the tone reference>
**Structural choice:** <what makes this variant distinct from the others>
**Rationale:** <1-2 sentences on why this works>

<post text>

---

## Variant 2: <short distinguishing label>

[same shape]

---

## Variant 3: <short distinguishing label>

[same shape]
```

**Also write to Obsidian** at `Writing/LinkedIn/Drafts/{topic-slug}_{YYYYMMDD}.md` (the same content). One file per invocation, all 3 variants sectioned within.

The topic slug is uppercase-snake or lowercase-kebab — pick whichever matches the user's existing files in `Writing/LinkedIn/Drafts/`. Default lowercase-kebab.

### Step 8 — Iteration within session

After producing the variants, the user may refine. Common refinements:

- "Make variant 2 more direct."
- "Shorten variant 1 by ~30%."
- "Drop the emoji from variant 3."
- "Combine variant 1's hook with variant 3's body."
- "Generate a variant 4 with a different angle."

Apply the refinement in-session. The skill remembers all prior drafts within the conversation, so the user doesn't re-paste. Update the Obsidian draft file with the refined content (overwrite the relevant variant section).

When the user is satisfied / done iterating, exit silently. The user manages publishing externally.

#### Question pacing

Minimal — only ask if something is genuinely blocking. Hard cap: 5 questions per turn. Most invocations should require 0-2 follow-up questions.

## What this skill does not do

- **Does not draft without anchoring.** No bundled default tone reference. Refuses if neither brand artifacts nor a tone reference exists.
- **Does not produce generic LinkedIn-style posts.** Hooks, structure, formatting, CTA presence — all governed by the tone reference. If the user wants generic, they can use a different tool.
- **Does not track publishing.** Draft tracking is out of scope. The user knows what they posted.
- **Does not maintain a per-topic history.** Each invocation produces a fresh set of variants; prior drafts stay in `Writing/LinkedIn/Drafts/` for the user to reference manually.
- **Does not commit anything.** Pure Obsidian write + chat output. Git is unaffected.
- **Does not invoke other skills automatically.** If brand artifacts or tone references are missing, the skill names the upstream skill (`/personal-brand-coach`, `/generate-tone-reference`) and exits.

## Output contract (the Obsidian draft file)

```yaml
---
topic: <topic>
generated: <YYYY-MM-DD>
audience: <inferred audience>
brand_source: <"AI/PersonalBrand/" or "(none)">
tone_reference: <path to tone reference used, or "(none)">
pillar_coverage: <list of pillars spoken to>
---
```

Body: the three variants as in the chat output (or N variants if iteration produced more).

## Examples

### Example 1 — clean draft with full anchoring

User: "/linkedin-post-synthesis on the EARS approach to acceptance criteria — theme is 'requirements engineering for vibes-coders'. Brain dump: I keep seeing AI-built apps with vague specs. EARS gives a structure. Most engineers don't know it. Worth surfacing."

Skill: reads `AI/PersonalBrand/` (3 pillars, audience: senior eng + founders), reads `Writing/ToneReferences/linkedin-personal.md` (terse declarative voice, sparse emoji, no CTA per tone). Topic maps cleanly to pillar "shipping with rigor". Audience: senior eng (the topic skews technical within the canonical audience).

Produces 3 variants:
- Variant 1 — story-led ("Last week I reviewed a spec that…")
- Variant 2 — contrarian-claim ("Most AI-built apps fail at the spec layer…")
- Variant 3 — practical-howto ("Three EARS patterns that fix vague AI-spec criteria")

Each with pillar mapping + rationale. Mirrored to `Writing/LinkedIn/Drafts/ears-acceptance-criteria_20260502.md`.

User: "Variant 2 is closest. Make it 30% shorter and drop the question at the end."

Skill: refines variant 2 in-session, updates the Obsidian draft file (overwrites variant 2 section, leaves 1 and 3 untouched).

### Example 2 — refusal: no anchoring

User: "/linkedin-post-synthesis on burnout"

Skill: detects no `AI/PersonalBrand/` and `_registry.md` shows no tone references. Refuses:

> No brand artifacts (`AI/PersonalBrand/`) and no tone references (`Writing/ToneReferences/`) exist. This skill needs at least one to draft posts in your voice — it doesn't ship with a fallback.
>
> Run `/personal-brand-coach` first (produces pillars / audience / positioning / a light voice anchor — covers both bases). Or, if you only want voice anchoring, run `/generate-tone-reference` against your prior writing. Then re-invoke this skill.

### Example 3 — off-brand request

User: "/linkedin-post-synthesis on cryptocurrency adoption — theme is 'web3 will replace traditional finance'"

Skill: cross-checks against `pillars.md` (none of the pillars cover crypto / finance). Cross-checks `anti-positioning.md`: contains "I will not opine on crypto / web3 / speculative tech." Surfaces:

> "This topic sits in your `anti-positioning.md` ('I will not opine on crypto / web3 / speculative tech.'). Two options: (a) deliberate deviation — proceed and add an anti-positioning v2 note explaining the shift, (b) re-think — pick a different angle (technical critique of an underlying mechanism vs. broad take on adoption)?"

Wait for the user's pick before drafting.

### Example 4 — partial anchoring

User invokes the skill in a fresh setup where `AI/PersonalBrand/` exists (recently bootstrapped via `/personal-brand-coach`) but no tone references have been generated yet.

Skill: proceeds with brand artifacts only. Warns: "No tone reference found. Drafts will use the personality / register cues in `voice-light.md` but won't have detailed structural patterns. Recommend running `/generate-tone-reference` against your existing writing for tighter voice match."

Produces 3 variants using `voice-light.md` as the voice anchor. The rationale section notes the partial anchoring.
