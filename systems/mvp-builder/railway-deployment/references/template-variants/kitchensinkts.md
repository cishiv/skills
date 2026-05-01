# kitchen-sink-ts — Railway deploy variant

**Snapshot date:** 2026-05-01
**Repo:** `github.com/cishiv/kitchen-sink-ts`

Single Bun process running TanStack Start with SSR. One Railway service + managed Postgres.

## Services

| Service | Role | Builder | Build command | Start command |
|---|---|---|---|---|
| `web` | The full app (SSR + API + static) | Nixpacks (Railway default) | `bun install && bun run build` | `bun run start` |

The repo already contains `railway.json` declaring this configuration:

```json
{
  "$schema": "https://railway.com/railway.schema.json",
  "build": {
    "builder": "NIXPACKS",
    "buildCommand": "bun install && bun run build"
  },
  "deploy": {
    "startCommand": "bun run start",
    "healthcheckPath": "/",
    "healthcheckTimeout": 300,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 5
  }
}
```

The skill should respect this — don't override the build/start commands or the health check path unless the user asks.

## Postgres

**Required.** Provision Railway's managed Postgres add-on at the smallest tier. Link it to the `web` service so `${{ Postgres.DATABASE_URL }}` becomes available as a reference variable.

## Domain

One dynamic Railway domain on the `web` service (e.g. `<project-name-lower>-web.up.railway.app`).

## Health check

- Path: `/` (per `railway.json`)
- Timeout: 300s (per `railway.json`)
- Port: Railway-assigned via `PORT` env var; the app must listen on `process.env.PORT`

## Post-deploy command

`bun run db:migrate`

Runs Drizzle migrations against the linked Postgres. Set on the `web` service via Railway MCP. If the MCP doesn't expose post-deploy command configuration, surface this command + service name to the user with instructions to set it in the Railway dashboard.

## Env vars (from `.env.example`)

| Var | Class | Source |
|---|---|---|
| `BETTER_AUTH_SECRET` | `USER_SET` | secret — user enters in dashboard |
| `BETTER_AUTH_URL` | `SKILL_SET` | derived: `https://<web-domain>` |
| `DATABASE_URL` | `SKILL_SET` | reference: `${{ Postgres.DATABASE_URL }}` |
| `POLAR_MODE` | `SKILL_SET` | literal: `production` (or template default) |
| `POLAR_ACCESS_TOKEN` | `USER_SET` | secret — only set if Polar is in scope per the spec |
| `POLAR_SUCCESS_URL` | `SKILL_SET` | derived: `https://<web-domain>/billing/success` (only if Polar is in scope) |
| `POLAR_WEBHOOK_SECRET` | `USER_SET` | secret — only if Polar in scope |
| `POLAR_THEME` | `SKILL_SET` | literal: template default |
| `R2_ACCOUNT_ID` | `SKILL_SET` | identifier (not a credential) — only if R2 in scope |
| `R2_ACCESS_KEY_ID` | `USER_SET` | secret — only if R2 in scope |
| `R2_SECRET_ACCESS_KEY` | `USER_SET` | secret — only if R2 in scope |
| `R2_BUCKET_NAME` | `SKILL_SET` | identifier — only if R2 in scope |
| `OPENROUTER_API_KEY` | `USER_SET` | secret — only if OpenRouter in scope |
| `OPENROUTER_DEFAULT_MODEL` | `SKILL_SET` | literal: from spec or template default |

**Integration scope.** If the MVP spec marked an integration as `REMOVE` or `NOT_PRESENT`, skip its env vars entirely (they're not needed and would confuse the runtime).

## Quirks

- `bun run start` runs `bun dist/server/server.js`. The build output path is load-bearing — don't change it without updating both `package.json` and `railway.json`.
- The app uses `bun --bun` wrapping in many scripts to force Bun's runtime over Node. Railway's Nixpacks builder uses Bun, so this is fine.
- `db:doc` is a stub command; not relevant for deploy.
