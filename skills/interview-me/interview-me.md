---
name: interview-me
description: "Interview the user with batched, themed questions written to Obsidian (CLAUDE_Q&A/) before producing any output. Use whenever the user invokes /interview-me, says \"ask me questions first\", or requests a complex deliverable (spec, system prompt, architecture doc, strategy, decision framework) without enough context to proceed confidently. Always use this skill before producing complex outputs when context is thin — don't guess, interview."
---
 
A structured intake skill. When invoked, Claude interviews the user by writing questions to Obsidian, waits for offline answers, reads them back, explicitly summarizes its understanding for confirmation, then proceeds with the deliverable.
 
The purpose of the interview is to surface the user's own thinking — not to have AI fill in the blanks. Questions go to Obsidian so the user can answer without AI assistance.
 
## Workflow
 
### Step 1: Clarify scope (in chat)
 
Before writing any questions, ask 2–4 brief scope questions inline in chat. These establish what's in and out of bounds so the interview stays focused and doesn't waste the user's time.
 
Examples:
 
- "Should I include questions about timeline and budget, or is this purely technical?"
- "Are stakeholders outside your team relevant here?"
- "Is this for an existing system or something new?"
- "Is the audience technical or non-technical?"
 
Wait for the user's answers before proceeding to Step 2.
 
---
 
### Step 2: Write questions to Obsidian
 
**Filename:** `CLAUDE_Q&A/{TOPIC}_Q&A.md`
 
Derive `{TOPIC}` from the user's request. Use underscores, no spaces (e.g. `System_Prompt_Q&A.md`, `Architecture_Review_Q&A.md`).
 
If the file already exists, append a new session block. If it doesn't, create it fresh.
 
**File structure:**
 
```markdown
### Session #{N} {DATE}
 
#### [Theme Name]
 
1. Question one?
   > *[your answer here]*
 
2. Question two?
   > *[your answer here]*
 
#### [Theme Name]
 
3. Question three?
   > *[your answer here]*
```
 
**Question writing principles:**
 
- Ask as many questions as needed upfront — err on the side of more
- Group by theme; themes vary by topic but common ones include: background/context, goals and outcomes, constraints, audience, technical details, edge cases, success criteria
- Within each theme, order broad to specific
- Don't ask about things the user has already told you
- Don't ask questions whose answers won't meaningfully change your output
- Don't assume scope — if something like timeline, budget, or stakeholders might be relevant, ask whether it's in scope rather than asking about it directly
 
**Session header:**
 
- `N` = increment from any existing sessions in the file; start at 1 for new files
- `DATE` = `DD Month YYYY` format (e.g. `13 April 2026`)
 
Use `obsidian_append_content` to write the file.
 
---
 
### Step 3: Notify the user (in chat)
 
After writing, tell the user:
 
- The file path
- How many questions across how many themes
- That they should answer in Obsidian without AI assistance, then come back
 
Example:
 
> "Written 21 questions to `CLAUDE_Q&A/System_Prompt_Q&A.md` across 5 themes. Go answer them in Obsidian — no AI — and let me know when you're done."
 
---
 
### Step 4: Read answers back
 
When the user returns, use `obsidian_get_file_contents` to read the file.
 
**Important:** Do not use `obsidian_batch_get_file_contents` — it is prone to timeouts. Always read single files with `obsidian_get_file_contents`.
 
---
 
### Step 5: Summarize understanding (in chat)
 
Before producing any output, write an explicit summary of what you understood. Structure:
 
- **Goal** — what the user is trying to achieve
- **Context and constraints** — the things that shape the approach
- **Assumptions** — anything you're inferring or that was ambiguous; flag these clearly
- **What you're about to produce** — the specific deliverable and its scope
 
Ask the user to confirm or correct before proceeding. Do not skip or abbreviate this step — its purpose is to clear out misinterpretations before effort is spent.
 
---
 
### Step 6: Produce the output
 
Once the user confirms your understanding is correct, proceed with the deliverable.
 
If answers were sparse or a critical question went unanswered, ask targeted follow-up questions in chat before proceeding. Keep follow-ups minimal and specific — don't re-interview.
 
---
 
## Tool reference
 
|Operation|Tool|
|---|---|
|Write questions to a new or existing file|`obsidian_append_content`|
|Read answers back|`obsidian_get_file_contents`|
|Check if a file already exists|`obsidian_list_files_in_dir`|
 
Load these tools via `tool_search` before using them if not already in context.
