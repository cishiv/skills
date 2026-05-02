---
name: generate-tone-reference
description: Use this skill when the user wants to extract a structured "tone reference" from a body of writing samples — their own posts, their blog, transcripts, anyone's writing they want to ape. Trigger on `/generate-tone-reference`, "extract a tone reference", "build a voice guide from these", "what does my writing sound like". Reads sources from Obsidian (paths, folders, lists), web pages (via `web_fetch`), and local repos with `.md` files (when on a surface with filesystem access). Produces `Writing/ToneReferences/{name}.md` in Obsidian — a hybrid doc with human-readable prose plus a machine-parseable frontmatter and structured sections downstream skills can consume. Maintains `Writing/ToneReferences/_registry.md` so other skills can discover available references. Works on any surface that has the Obsidian MCP.
---

# generate-tone-reference

Read a body of writing samples → produce a structured tone reference doc that captures voice attributes, vocabulary, structural patterns, do/don't guardrails, and a small set of anchor exemplars.

The output is consumed by writing-synthesis skills (`/linkedin-post-synthesis` and others to come) so they can write *in your voice* rather than in a generic LLM voice. The skill is largely deterministic — feed it sources, get a doc — so the workflow is short and the questions are minimal.

## Boundaries

- **Any surface that has the Obsidian MCP.** Output lives in Obsidian; the skill needs to write there. Local-repo `.md` ingestion requires filesystem access (Claude Code).
- **Sources can be any combination** of Obsidian paths (file, folder, list), web pages (via `web_fetch`), and local repos with `.md` files. Pasted text in chat is allowed for one-off / quick references that don't get persisted as part of the source set.
- **Single-batch interactive.** No iterative deep-dive. The skill ingests, synthesizes, writes, and exits in one turn (with a single optional question batch up front if input is ambiguous). Question cap: none — only ask what's actually needed.
- **More sources is better.** Skill works with whatever's given but warns when sample size is low. No hard floor.
- **Multi-author content gets flagged, not excluded.** If a sample is clearly written by someone else (a quoted article, a guest's transcript), the skill still ingests but flags it; the user decides whether to keep or drop in a follow-up if they care.
- **Updates with the same name require explicit user direction.** Don't silently regenerate or merge. Refuse + ask.
- **Maintains a registry.** `Writing/ToneReferences/_registry.md` is updated every time a reference is produced or removed. Downstream skills read the registry to discover available references.

## Workflow

### Step 1 — Ingest sources

Ask the user (if not provided) for the source set. Acceptable shapes (any combination):

- **Obsidian path** — single file (e.g. `Writing/Blog/post-2026-04-15.md`) or folder (e.g. `Writing/Blog/` — skill loads all `.md` inside).
- **List of paths** — e.g. `["Writing/Blog/", "Writing/Talks/keynote-2025.md"]`.
- **Web page** — URL the skill fetches via `web_fetch`.
- **Local repo** (Claude Code surface only) — directory path; skill globs all `.md` files within.
- **Pasted text** — for quick / scratch references that don't persist as a source citation.

For each source, capture: the path/URL, the text content, and (if Obsidian or local file) the modification date.

#### Length filtering

Include everything. Don't discard short samples — every sample contributes some signal.

#### Multi-author detection

Scan each sample for clear multi-author markers (block quotes, "as <name> said", explicit attribution). When detected, flag the sample as `multi-author` in the sources appendix. Don't exclude; the user can decide whether to drop later.

### Step 2 — Confirm naming and intended use

Propose a reference name from the sources. Heuristics, in order:

1. If sources are from a single Obsidian folder, use the folder name (e.g. `Writing/Blog/` → `blog`).
2. If sources span multiple paths but share a topic / theme (detected by repeated terms), use that.
3. Fall back to a date-stamped placeholder (`reference_YYYYMMDD`) and ask the user to rename.

Confirm with the user: "I'll call this `<name>` — change?"

Ask the **intended use** in one short question: e.g. `linkedin-posts`, `blog-posts`, `talks`, `general-writing`. This goes in the frontmatter. If the user doesn't have a strong opinion, default to `general-writing`.

#### Same-name collision

If `Writing/ToneReferences/{name}.md` already exists, **ask the user what to do**: rename the new one, overwrite, or abort.

### Step 3 — Extract tone

Synthesize across the sample set. Produce structured findings for each section (see Step 4 for the exact output shape):

- **Voice attributes** — tone, register, perspective (first / second / third person; declarative / inquisitive), energy, signature mood.
- **Vocabulary** — phrases used often (with frequency or "common" / "rare"), characteristic words, words avoided (gaps that suggest a deliberate avoidance).
- **Structural patterns** — typical openings (hooks, throat-clearing, in medias res), transitions, closings, paragraph length, sentence length distribution.
- **Do / Don't list** — explicit guardrails the writing follows. "Avoids hedging." "Never uses corporate-speak." "Always concrete examples." Each one a single-sentence rule.
- **Anchor exemplars** — 3-5 short snippets (1-3 sentences each) that best embody the voice. Choose ones that downstream synthesis can pattern-match against.

#### Voice coherence check

If the samples span clearly different voices (different registers, different audiences, different intents), surface to the user: "Sources span at least N distinct voices — group A reads as <description>, group B reads as <description>. Pick one to extract, or proceed with a 'mixed' reference?" Show which samples land in each group.

#### Low-confidence warning

If sample size is below a meaningful floor (heuristic: <3 distinct samples, or total wordcount <2000), produce the reference but flag in the frontmatter `confidence: "low"` and include a top-of-file note recommending more sources.

### Step 4 — Write the tone reference

**Output path:** `Writing/ToneReferences/{name}.md`

**Frontmatter:**

```yaml
---
name: <name>
generated: <YYYY-MM-DD>
source_count: <N>
intended_use: <linkedin-posts | blog-posts | talks | general-writing | ...>
confidence: <high | medium | low>
multi_author_flagged: <true | false>
---
```

**Body sections** (in this order):

1. **Voice attributes** — prose paragraphs (1-3) covering tone, register, perspective, energy, signature mood. Human-readable.
2. **Vocabulary** — two sub-sections: `Used often` (bulleted list of phrases / words with brief context) and `Avoided` (bulleted list with brief rationale per gap).
3. **Structural patterns** — sub-sections for `Openings`, `Transitions`, `Closings`. Each a short prose paragraph + 1-2 representative examples.
4. **Do / Don't** — two-column table or two bulleted lists. Each entry a single-sentence rule.
5. **Anchor exemplars** — 3-5 fenced blocks, each an exemplar snippet. Format:
   ```
   > <snippet text>
   ```
   No source citation per snippet (Q9: stay clean).
6. **Sources appendix** — bulleted list of every source path/URL ingested, with any `multi-author` flags noted inline.

The body is human-readable prose / lists; the frontmatter and the section structure (consistent across all tone references) is what makes it machine-consumable for downstream skills.

### Step 5 — Update the registry

Read `Writing/ToneReferences/_registry.md`. If it doesn't exist, create it.

Registry shape:

```markdown
# Tone references registry

Maintained by `/generate-tone-reference`. Lists all available tone references in `Writing/ToneReferences/`. Downstream skills read this to discover what's available.

| Name | Intended use | Generated | Source count | Confidence |
|---|---|---|---|---|
| blog | blog-posts | 2026-05-02 | 12 | high |
| linkedin-personal | linkedin-posts | 2026-05-02 | 8 | medium |
```

Append the new reference's row (or update an existing row if it was overwritten in Step 2).

### Step 6 — Output to user

In chat:

- The reference name and Obsidian path.
- Source count + confidence flag.
- Whether multi-author content was flagged.
- A note that the registry was updated.

Don't dump the full reference content into chat — the file is the deliverable.

## What this skill does not do

- **Does not score writing quality.** The reference describes the voice as-is; no judgment about whether it's good.
- **Does not modify source files.** Read-only against everything ingested.
- **Does not silently regenerate.** Same-name collision → ask the user.
- **Does not cite sources per pattern in the body.** Sources go in an appendix only; the main body stays clean for downstream skills to consume.
- **Does not auto-invoke other skills.** If the produced reference is intended for `/linkedin-post-synthesis`, the user invokes that skill themselves.

## Frontmatter contract (produced)

| Field | Value |
|---|---|
| `name` | The reference name (Obsidian-path-safe, lowercase-snake or kebab) |
| `generated` | YYYY-MM-DD the reference was produced |
| `source_count` | Number of distinct sources ingested |
| `intended_use` | Free-text label, default `general-writing` |
| `confidence` | `high` / `medium` / `low` based on sample size + coherence |
| `multi_author_flagged` | `true` if any source was flagged as multi-author |

## Examples

### Example 1 — clean blog ingestion

User: "/generate-tone-reference for my blog at Writing/Blog/"

Skill: loads all `.md` files in `Writing/Blog/` (12 posts), all single-author, total ~18k words. Confirms name `blog`, intended use `blog-posts`. Synthesizes the reference. Writes `Writing/ToneReferences/blog.md` with `confidence: high`. Updates `_registry.md`.

Chat output: "`Writing/ToneReferences/blog.md` written (12 sources, confidence: high). Registry updated."

### Example 2 — mixed-voice surfacing

User: "/generate-tone-reference for everything in Writing/"

Skill: loads `Writing/Blog/` (12 posts, casual reflective voice), `Writing/Talks/` (3 transcripts, formal stage-y voice), `Writing/LinkedIn/` (8 posts, terse declarative voice). Surfaces: "Three distinct voices detected. Group A (blog, 12 samples): casual reflective. Group B (talks, 3 samples): formal. Group C (LinkedIn, 8 samples): terse. Pick one or proceed with mixed?"

User: "Just blog and LinkedIn — they're closer."

Skill: re-runs on the subset, produces `Writing/ToneReferences/written-shiv.md` with `confidence: high`. Updates registry.

### Example 3 — low confidence

User: "/generate-tone-reference for my one substack post"

Skill: loads the single source. Produces the reference with `confidence: low` and a top-of-file note: "Reference built from 1 source, ~800 words. Recommend adding more samples to improve coverage."

### Example 4 — same-name collision

User: "/generate-tone-reference for my recent LinkedIn posts" (where `Writing/ToneReferences/linkedin-personal.md` already exists from a prior run).

Skill: surfaces the collision: "`linkedin-personal.md` already exists (generated 2026-04-20, 5 sources). Options: (a) rename this run, (b) overwrite the existing, (c) abort."

User picks (b). Skill overwrites + updates the registry row.
