# Template capabilities (bundled fallback)

This directory contains snapshots of what each supported template repo provides for free. The skill uses these **only when the live template can't be reached**.

## When to use which source

The skill follows this chain to acquire template capabilities, in order:

1. **Live read** — if the target repo is cloned and accessible (Claude Code with a working directory inside the repo), read `CLAUDE.md` (especially the agent-writable Reference section), `DATABASEMODEL.md` if present, and the scaffold tree. This is the source of truth.
2. **GitHub MCP** — if connected, fetch `CLAUDE.md` from the template repo's `main` branch.
3. **`web_fetch`** — if available and the repo is public, fetch `https://raw.githubusercontent.com/cishiv/<template>/main/CLAUDE.md`.
4. **Bundled snapshot (this directory)** — last resort. **Tell the user explicitly** which snapshot date is in use and recommend they verify against the live template before committing.

The bundled snapshots are concise summaries — they capture what's PROVIDED, CONFIGURABLE, and NOT_PRESENT, plus the essential file map and commands. They are **not** a substitute for the live `CLAUDE.md`; the live document has fuller recipes and gotchas the skill doesn't need at scope-cutting time.

## Files

- `kitchen-sink-ts.md` — single SSR Bun + TanStack Start deployable.
- `kitchen-sink-twotier.md` — separate Bun + Hono backend + React + Vite client + Zod-shared workspace.
- `statix.md` — static markdown site generator (Preact + Vite + nginx).

Each file includes a snapshot date in its header.

## When this directory needs updating

When any of the templates ships a meaningful change — new integration scaffolded in, an existing one removed, a major framework upgrade. Re-snapshot the affected file. The user is responsible for keeping this directory in sync; the skill cannot detect drift on its own.
