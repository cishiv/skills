# Template contracts (bundled fallback)

This directory contains snapshots of `SPECIFICATIONS/SPEC_TEMPLATE.md` for each supported template repo. The skill uses these **only when the live template can't be reached**.

## When to use which source

The skill follows this chain to acquire the spec template, in order:

1. **Filesystem** — if the target repo is cloned and accessible (typical Claude Code with a working directory inside the repo), read `SPECIFICATIONS/SPEC_TEMPLATE.md` directly. This is the source of truth.
2. **GitHub MCP** — if the GitHub MCP is connected, fetch the file from the repo's `main` branch.
3. **`web_fetch`** — if `web_fetch` is available and the repo is public, fetch the raw URL: `https://raw.githubusercontent.com/cishiv/<template>/main/SPECIFICATIONS/SPEC_TEMPLATE.md`.
4. **Bundled snapshot (this directory)** — last resort. **Tell the user explicitly** which snapshot date is in use and recommend they verify against the live template before committing.

The bundled snapshot is sufficient to produce a structurally valid spec — frontmatter, sections, statix-specific differences. But it can drift. The user is the safety net against drift; Claude must surface the use of the fallback explicitly.

## Files

- `kitchen-sink-ts.md` — snapshot for the SSR single-deployable template.
- `kitchen-sink-twotier.md` — snapshot for the Bun + Hono + React + workspaces template.
- `statix.md` — snapshot for the static markdown site generator.

Each file includes a snapshot date in its header and the literal template content in a code fence.

## When this directory needs updating

When `SPEC_TEMPLATE.md` changes in any of the three repos. Re-snapshot the affected templates here after any template amendment. The user is responsible for keeping this directory in sync with the templates; the skill cannot detect drift on its own.
