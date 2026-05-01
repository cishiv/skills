# kitchen-sink-twotier — Railway deploy variant

**Snapshot date:** 2026-05-01
**Repo:** `github.com/cishiv/kitchen-sink-twotier`

Bun + Hono backend (workspace `server/`) and Vite + React SPA (workspace `client/`). Two Railway services + managed Postgres. Cross-domain coordination via env vars.

## Services

| Service | Role | Builder | Build command | Start command |
|---|---|---|---|---|
| `server` | Hono backend (API + auth + DB) | Nixpacks | `bun install` (workspaces) | `bun run --filter './server' start` (or `cd server && bun run start`) |
| `client` | Vite SPA (built static, served by Railway's static hosting or a static-server) | Nixpacks | `bun install && bun run --filter './client' build` | static — Railway serves `client/dist/` |

The twotier repo does **not** carry a `railway.json` — both services are configured via Railway MCP at provisioning time.

**Notes on the client service.** The client is a Vite SPA — output is static `client/dist/`. Railway can serve this via:
- A static-only service pointing at `client/dist/` (preferred — no runtime cost).
- Or a thin static-file server if Railway's static hosting requires it.

The skill should consult `docs.railway.com/llms.txt` (Step 5) for the current right shape — Railway's static hosting capabilities have evolved.

## Postgres

**Required.** Provision Railway's managed Postgres at the smallest tier. Link to the `server` service. `${{ Postgres.DATABASE_URL }}` becomes available as a reference variable on `server`.

## Domains

Two dynamic Railway domains:

- `<project-name>-server.up.railway.app` for `server`.
- `<project-name>-client.up.railway.app` for `client`.

The two domains must reference each other via env vars (server allows the client's origin via `TRUSTED_ORIGINS`; client knows where to reach the API).

## Health check

- Path: `/` on each service.
- Server: HTTP 200 from Hono root.
- Client: HTTP 200 from Vite's `index.html`.
- Port: Railway-assigned via `PORT`. Server must listen on `process.env.PORT`. Client static-serve uses Railway's default.

## Post-deploy command

`bun run --filter './server' db:migrate`

Set on the `server` service via Railway MCP. If MCP can't expose post-deploy commands, surface this command + service name to the user.

If the monorepo workspace `--filter` syntax doesn't work in Railway's deploy environment for some reason, fall back to `cd server && bun run db:migrate` and surface the change to the user.

## Env vars (from `server/.env.example`)

### Server service

| Var | Class | Source |
|---|---|---|
| `PORT` | `SKILL_SET` | literal: `3000` (Railway will override with its assigned port) |
| `NODE_ENV` | `SKILL_SET` | literal: `production` |
| `DATABASE_URL` | `SKILL_SET` | reference: `${{ Postgres.DATABASE_URL }}` |
| `BETTER_AUTH_SECRET` | `USER_SET` | secret |
| `BETTER_AUTH_URL` | `SKILL_SET` | derived: `https://<server-domain>` |
| `TRUSTED_ORIGINS` | `SKILL_SET` | derived: `https://<client-domain>` |
| `POLAR_ACCESS_TOKEN` | `USER_SET` | secret — only if Polar in scope |
| `POLAR_SERVER` | `SKILL_SET` | literal: `production` (only if Polar in scope) |
| `POLAR_WEBHOOK_SECRET` | `USER_SET` | secret — only if Polar in scope |
| `POLAR_SUCCESS_URL` | `SKILL_SET` | derived: `https://<client-domain>/billing/success` |
| `OPENROUTER_API_KEY` | `USER_SET` | secret — only if OpenRouter in scope |
| `R2_ACCOUNT_ID` | `SKILL_SET` | identifier — only if R2 in scope |
| `R2_ACCESS_KEY_ID` | `USER_SET` | secret — only if R2 in scope |
| `R2_SECRET_ACCESS_KEY` | `USER_SET` | secret — only if R2 in scope |
| `R2_BUCKET` | `SKILL_SET` | identifier — only if R2 in scope |

### Client service

The client's env vars are `VITE_`-prefixed and bundled into the static build. Set at build time, not runtime.

| Var | Class | Source |
|---|---|---|
| `VITE_API_URL` (or equivalent — confirm at runtime) | `SKILL_SET` | derived: `https://<server-domain>` |

If the client uses additional `VITE_*` vars, classify them per the same heuristic — but most should be `SKILL_SET` since they're public (anything in `VITE_` is in the client bundle).

**Integration scope.** Skip env vars for integrations marked `REMOVE` or `NOT_PRESENT` in the MVP spec.

## Quirks

- The server runs from source (`bun run src/index.ts`) — there is no separate build step for the server workspace. Bun executes TypeScript directly. This means the deploy build is just `bun install`; there's no precompiled output to ship.
- The client builds via `bunx --bun tsc -b && bunx --bun vite build` (TypeScript build + Vite build).
- The two services don't share a domain — cross-origin communication is intentional. The `TRUSTED_ORIGINS` env var on the server explicitly allows the client domain.
- `auth:generate` is a one-off command for regenerating BetterAuth's schema; not relevant for deploy.
- The server uses Biome (not ESLint+Prettier) for formatting/linting. Doesn't affect deploy.
