# statix — Railway deploy variant

**Snapshot date:** 2026-05-01
**Repo:** `github.com/cishiv/statix`

Static Preact + Vite SPA built into `dist/` and served by nginx. Single Railway service via Dockerfile. No database, no env vars, no migrations.

## Services

| Service | Role | Builder | Build command | Start command |
|---|---|---|---|---|
| `web` | Static site (nginx serving Preact build) | **Dockerfile** | (Dockerfile handles build) | (Dockerfile `CMD` — nginx) |

The repo carries a `Dockerfile`:

```dockerfile
FROM oven/bun:1-alpine AS build
WORKDIR /app
COPY . .
RUN bun install --frozen-lockfile && bun run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
```

Configure the Railway service to use the **Dockerfile builder**, not Nixpacks. Path: `Dockerfile` (root of repo).

## Postgres

**Not required.** Statix has no DB. Skip Postgres provisioning entirely.

## Domain

One dynamic Railway domain on `web` (e.g. `<project-name>-web.up.railway.app`).

## Health check

- Path: `/` (nginx serves `index.html`)
- Port: **8080** (per `nginx.conf` `listen 8080`)

The Railway service must be configured to expose port 8080, not the typical Railway-default port. Set this explicitly via MCP at provisioning time.

## Post-deploy command

**None.** Statix has no migrations or post-deploy steps.

## Env vars

**None.** Statix has no `.env.example`, no runtime env, no integrations.

If the user has somehow introduced env vars into a statix-based project, the skill should surface them as a deviation from the template (per Step 7 refusal-equivalent — surface but don't refuse).

## Quirks

- **Port mismatch with Railway defaults.** Railway typically expects services to listen on `$PORT`, but nginx in this template is hardcoded to 8080. Configure the Railway service's port to 8080 explicitly. If this becomes a maintenance pain, a future template amendment could parameterize nginx's listen directive — but that's outside this skill's scope.
- **Bun is only used during build.** Runtime is nginx (alpine). No Bun in the production image — keep this in mind if a future feature needs server-side logic, which would require a different template.
- **Vitest runs under Node, not Bun** in this template (per `CLAUDE.md`). Doesn't affect deploy — tests aren't run in production.
- **No GitHub Actions or CI** in the template. Railway's built-in build is the only build path.
- **Wikilinks are resolved at build time.** Broken wikilinks surface as warnings, not errors — they don't fail the build. The deploy will succeed even with broken wikilinks; surface them in the deploy report if `bun run build` output mentions them.
