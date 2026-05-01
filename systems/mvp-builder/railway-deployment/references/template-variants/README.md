# Template variants (deploy-time differences)

This directory contains per-template snapshots of the deploy-time configuration each template needs on Railway. The skill loads the file matching the detected template (Step 2 of the workflow) and uses it to drive provisioning.

## Files

- `kitchensinkts.md` — single Bun + TanStack Start service + Postgres.
- `twotier.md` — two services (Hono server + Vite SPA client) + Postgres.
- `statix.md` — single Dockerfile-based service (Preact build behind nginx). No DB.

Each file declares:

- **Services** — count, role, builder type (Nixpacks vs Dockerfile).
- **Build / start commands** per service.
- **Postgres requirement** — yes / no, and which service it links to.
- **Expected env vars** from the template's `.env.example`, classified as `SKILL_SET` (skill configures automatically) or `USER_SET` (user enters via Railway dashboard because it's a secret).
- **Health check** — path and expected port.
- **Post-deploy command** — for templates that need migrations.
- **Quirks** — anything template-specific worth surfacing.

## When this directory needs updating

Whenever a template's deploy shape changes — a new service required, a new env var added to `.env.example`, a build/start command change, a Dockerfile change, etc. Re-snapshot the affected file. The user is responsible for keeping this in sync with the templates; the skill cannot detect drift on its own.

## Source of truth

The bundled snapshots are concise — they capture deploy-relevant information only. The full template state lives in each template repo's `CLAUDE.md`, `package.json`, `railway.json` (where present), and `.env.example`. The skill cross-references the live template files at runtime when it's running in a target repo.

`docs.railway.com/llms.txt` is read fresh at runtime via `web_fetch` — Railway behavior changes outside our control, so it isn't bundled here.
