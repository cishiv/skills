---
name: railway-deployment
description: Use this skill to provision a fresh Railway project and execute the first deploy for a repo built from one of the three templates (kitchen-sink-ts, kitchen-sink-twotier, statix). Trigger on phrases like "deploy to Railway", "ship it", "first deploy", "set up Railway". Single parameterized skill — detects the template from the repo and applies template-specific provisioning. Project mode only (feature-mode "deploy" is just `git push`, which is outside this skill's scope). Claude Code only. Requires the Railway MCP and `web_fetch`. Refuses on dirty tree, non-main branch, failed local build, or already-linked Railway project.
---

# railway-deployment

This skill provisions a new Railway project and executes the first deploy for a repo built from one of the three templates. It runs end-to-end: Railway project + services + Postgres (where needed) + GitHub linkage + env vars + post-deploy migration command + first deploy + health check.

The skill is **single parameterized**, not three per-template skills. Per-template differences (services, build commands, Postgres need, Dockerfile path, env var shape) live in bundled `references/template-variants/{template}.md`.

The skill is designed to run **autonomously**. It only asks the user when something has materially deviated from the template (a new required service, a missing env var the template doesn't know about, a deploy failure that needs human triage). Otherwise it provisions, deploys, verifies, reports, and exits.

## Boundaries

- **Project mode only.** First-time provisioning + first deploy for a new Railway project. Subsequent deploys are `git push` (Railway picks up the change), which is outside this skill's scope.
- **Claude Code only.** No claude.ai surface. Real provisioning, real deploys.
- **Railway MCP required.** Not the CLI. The skill checks for the MCP server in tool availability and refuses if absent.
- **`web_fetch` required.** `docs.railway.com/llms.txt` is read at runtime to ground the skill in current Railway behavior. No bundled fallback.
- **Skill never touches secrets.** It does not read local `.env`, does not introspect API keys, does not echo secret values. Sensitive env vars are surfaced to the user for manual entry via the Railway dashboard.
- **Skill does not run migrations from local against Railway Postgres.** Migrations are configured as a Railway *post-deploy command* on the service via MCP. If the MCP can't set it, the skill surfaces the command for the user to configure manually.
- **Spec lifecycle check is a build, not a file lookup.** Whether or not `IMPLEMENTED_MVP_*.md` exists, the gate is: does the app build successfully locally? If yes, deployable. If no, refuse.

## Workflow

### Step 1 — Pre-flight

Validate environment and repo state. All checks are hard gates — refuse on any failure.

**Environment checks:**

- Railway MCP server is available in the tool list. Try a no-op MCP call (e.g. list projects) to confirm it responds.
- `web_fetch` tool is available.

**Repo state checks:**

- The user is on `main`.
- The working tree is clean (no uncommitted changes, no untracked files in `src/`, `package.json`, etc. — gitignored files like `node_modules/`, `dist/`, `.env` don't count).
- `git remote -v` shows a GitHub remote (`origin` pointing at `github.com:*`). Required because the deploy mechanism is `git push` to a Railway-linked GitHub repo.

**Build check (the real lifecycle gate):**

- Run `bun install` (idempotent if already installed).
- Run `bun run build`. Production build must succeed. For statix this produces `dist/`; for kitchen-sink-ts it produces `dist/server/server.js`; for twotier it builds both workspaces.
- If the build fails, refuse with the build output. The user fixes the build before re-invoking.

#### Refusal: any pre-flight check fails

Two-paragraph refusal. First: name the specific failure (Railway MCP missing, dirty tree, build failed, etc.). Second: tell the user the concrete next action (install Railway MCP, commit/stash changes, fix build errors X, etc.). Don't try to remediate — refuse cleanly.

### Step 2 — Detect template and load variant reference

Read the repo's `CLAUDE.md`. The first heading declares the template (e.g. `# kitchen-sink-ts — Ways of Working`).

Match against one of: `kitchen-sink-ts`, `kitchen-sink-twotier`, `statix`. If no match, refuse — the skill only supports these three templates.

Load the corresponding bundled reference: `references/template-variants/{kitchensinkts,twotier,statix}.md`. This file declares:

- Services to provision (one or more, with roles).
- Build and start commands per service.
- Whether Postgres is needed.
- Dockerfile path (statix) vs Nixpacks (others).
- Expected env vars (from `.env.example`), classified as `SKILL_SET` or `USER_SET` (sensitive).
- Health check path and port.
- Post-deploy command (migrations) for services that need it.

### Step 3 — Verify spec mode

Look for an MVP spec in `SPECIFICATIONS/IMPLEMENTED/` or `SPECIFICATIONS/NOT_YET_IMPLEMENTED/`. Read its frontmatter `mode`.

- `mode: project` → proceed.
- `mode: feature` → refuse. Tell the user this skill is project-mode only; for feature deploys, just `git push` to the existing Railway-linked GitHub repo.
- No MVP spec found → proceed if the build check passed (the app exists and works), but warn the user that there's no spec lifecycle artifact tying this deploy to a specification.

### Step 4 — Check Railway state for the repo

Via MCP, check if a Railway project is already linked to this GitHub repo.

- **No Railway project for this repo** → proceed to Step 5 (provisioning).
- **Railway project already exists** → refuse. Tell the user: this repo already has a Railway project linked. Subsequent deploys are `git push`. If they want to tear down and re-provision, do it via the Railway dashboard first, then re-invoke this skill.

### Step 5 — Read live Railway docs

Via `web_fetch`, fetch `https://docs.railway.com/llms.txt`. Use the contents to ground the skill's understanding of current Railway behavior — particularly anything that's changed since the bundled template-variant references were written.

If `web_fetch` succeeds, proceed. If it fails (network, 404, etc.), refuse — the skill is not allowed to provision against stale knowledge of Railway behavior.

### Step 6 — Provision the Railway project

Using the MCP, in this order:

1. **Create the Railway project.** Name it after the spec's `name` field (uppercase snake_case from the spec). Region: `EU` (default). If the user has a preference declared elsewhere, honor it.
2. **Create services per the template-variant reference.**
   - **kitchen-sink-ts**: one service (`web`).
   - **kitchen-sink-twotier**: two services (`server`, `client`). Each has its own build/start command and dynamic Railway domain.
   - **statix**: one service (`web`), Dockerfile-based.
3. **Provision Postgres** (kitchen-sink-ts and twotier only). Smallest tier by default.
4. **Link Postgres to the relevant service.** kitchen-sink-ts: `web`. twotier: `server`. This makes Railway-managed reference variables available.
5. **Provision a dynamic Railway domain** for each service that needs public access. kitchen-sink-ts: `web` only. twotier: both `server` and `client`. statix: `web` only.
6. **Configure GitHub repo linkage.** Connect the Railway project to the existing GitHub repo so `git push` triggers deploys. If the Railway GitHub app isn't installed in the user's GitHub account/org, surface the install URL and pause for the user to authorize, then continue.

### Step 7 — Configure env vars

Read the template's `.env.example` (or, for twotier, `server/.env.example`).

For each env var, classify:

- **`SKILL_SET`** — non-sensitive, has a sensible default OR is a Railway reference variable. Skill sets directly.
  - `PORT`, `NODE_ENV`, `BETTER_AUTH_URL`, `TRUSTED_ORIGINS`, `POLAR_MODE`, `POLAR_SERVER`, `POLAR_SUCCESS_URL`, `POLAR_THEME`, `OPENROUTER_DEFAULT_MODEL`, `R2_BUCKET_NAME` / `R2_BUCKET`, `R2_ACCOUNT_ID` (account ID is not a credential).
  - `DATABASE_URL` → set to `${{ Postgres.DATABASE_URL }}` (Railway reference variable).
  - `BETTER_AUTH_URL` → set to the dynamic Railway domain of the service it belongs to (e.g. `https://<server>.up.railway.app`).
  - For twotier client: derive `VITE_API_URL` (or equivalent) from the server's Railway domain.

- **`USER_SET`** — sensitive (matches secret patterns: `*_KEY`, `*_SECRET`, `*_TOKEN`, `*_PASSWORD`, `*_WEBHOOK_SECRET`, `*_ACCESS_TOKEN`). Skill does NOT set. Skill surfaces the variable name to the user with a note: "Add this in the Railway dashboard before the first deploy can succeed."

For each `USER_SET` var, write the var name (NOT the value — skill never reads or echoes secrets) into the deploy report (Step 11) so the user has a checklist.

#### Refusal: env var in code missing from `.env.example`

Skill scans the codebase for `process.env.X` / `import.meta.env.X` references. If any are NOT in `.env.example`, surface them to the user. The skill does NOT refuse outright — proceed with deploy, but note in the report that the runtime may fail on the missing var.

### Step 8 — Configure post-deploy command (migrations)

For kitchen-sink-ts and twotier, set the service's post-deploy command via MCP:

- **kitchen-sink-ts**: `bun run db:migrate` on the `web` service.
- **twotier**: `bun run --filter './server' db:migrate` on the `server` service. (Or `cd server && bun run db:migrate` — whichever the live `docs.railway.com/llms.txt` indicates is the right shape for monorepo workspaces.)

If the MCP doesn't expose post-deploy command configuration, surface the exact command and the service name to the user with instructions to set it in the Railway dashboard's service settings before the first deploy.

statix has no migrations — skip this step.

### Step 9 — Trigger first deploy

Push to the GitHub repo's `main` branch. Railway picks up the push and starts the deploy.

If the user's local `main` is already in sync with `origin/main`, no push is needed — Railway will deploy from the existing HEAD. Trigger via MCP if Railway exposes a "deploy current commit" call; otherwise the deploy will start once the GitHub app webhook arrives.

### Step 10 — Wait for deploy and health-check

Poll the Railway deployment state via MCP until the deploy completes (success or failure). Stream key state transitions to chat (build started, build succeeded, deploying, deployed) but don't dump full logs unless they're useful.

On **deploy success**, hit the public URL with an HTTP GET:

- kitchen-sink-ts: `GET https://<web>.up.railway.app/` (per `railway.json`'s `healthcheckPath: "/"`).
- twotier server: `GET https://<server>.up.railway.app/` (or whatever the Hono root returns).
- twotier client: `GET https://<client>.up.railway.app/` (Vite SPA index).
- statix: `GET https://<web>.up.railway.app/` (nginx index).

Expect HTTP 200. Anything else is a failure (4xx / 5xx / network error / timeout).

#### Refusal-equivalent: deploy fails or health check fails

Surface the failure logs from Railway. **Do not retry.** Tell the user the next action: typically inspect the logs, fix the issue (likely a missing `USER_SET` env var, a missing dependency, or a build issue that didn't show up in local `bun run build`), and re-invoke after fixing.

### Step 11 — Write the end-of-deploy report

Write `AGENT_REPORTS/DEPLOY_REPORT_{YYYYMMDD}_{HHMM}.md` with:

```yaml
---
report_type: "deploy"
template: "<kitchen-sink-ts | kitchen-sink-twotier | statix>"
project_name: "<spec.name>"
railway_project_id: "<id>"
deploy_started: "<ISO 8601>"
deploy_completed: "<ISO 8601>"
result: "<success | failure>"
---
```

Body sections:

- **Summary** — one paragraph: what was provisioned, what was deployed, what's pending user action.
- **Services** — list per service: name, role, public URL (Railway domain), build command, start command.
- **Postgres** — for kitchen-sink-ts and twotier: connection method (`${{ Postgres.DATABASE_URL }}` reference), tier, post-deploy migration command.
- **Env vars set by skill (`SKILL_SET`)** — list of var names + brief description of value source (literal | reference | derived from domain). Values omitted from the report.
- **Env vars pending user action (`USER_SET`)** — checklist of var names + Railway dashboard URL. The user must set these before the deploy can fully succeed (or before the next deploy if they were missing for the first one).
- **Env vars referenced in code but missing from `.env.example`** — if any.
- **Health check** — URL, response status, response time.
- **Failures** (if any) — full Railway log excerpts and suggested remediation.
- **Suggested next actions** — set the pending user-action env vars, verify the app behavior at the public URL, monitor next deploy on `git push`.

**Do not commit this file.** Same as `/build-from-spec` — designed for future HITL hooks.

### Step 12 — Final summary to user

In the chat:

- One line: deploy result, public URL.
- Any pending `USER_SET` env vars to configure (just the count + report path; the report has the full list).
- Path to the deploy report.

Don't ask "anything else?" Don't restate what's in the report.

## What this skill does not do

- **Does not run on claude.ai chat.** Claude Code only.
- **Does not work without Railway MCP.** Refuses cleanly. The user installs it and re-invokes.
- **Does not work without `web_fetch`.** Refuses cleanly.
- **Does not handle feature mode.** Subsequent deploys are `git push`. This skill exits if invoked against a feature-mode spec.
- **Does not touch secrets.** Skill never reads `.env`. Sensitive vars (matched by name pattern) are surfaced for user manual entry.
- **Does not run migrations from local.** Configures Railway post-deploy command instead.
- **Does not retry on deploy failure.** Surface logs, exit. Re-invocation after the user fixes the root cause.
- **Does not provision custom domains.** Default `*.up.railway.app` only.
- **Does not commit the deploy report.** User commits if they want.
- **Does not auto-suggest next skills.** Stays silent at the end.

## Sensitive-var detection heuristic

A var is `USER_SET` (sensitive) if its name matches any of:

- `*_KEY` (e.g. `OPENROUTER_API_KEY`, `R2_ACCESS_KEY_ID`)
- `*_SECRET` (e.g. `BETTER_AUTH_SECRET`, `POLAR_WEBHOOK_SECRET`)
- `*_TOKEN` (e.g. `POLAR_ACCESS_TOKEN`)
- `*_PASSWORD`
- `*_ACCESS_KEY*` (covers `R2_ACCESS_KEY_ID` and `R2_SECRET_ACCESS_KEY`)
- `*_WEBHOOK_SECRET`

Everything else from `.env.example` that isn't covered by Railway reference variables is `SKILL_SET` with the value carried over from `.env.example` (which contains placeholder defaults, not actual secrets — that's why `.env.example` is committed and `.env` is gitignored).

If a var is ambiguous (e.g. `BETTER_AUTH_URL` could be considered sensitive but it's actually a public URL), the skill defaults to `SKILL_SET` with the derived value. The user can override via the Railway dashboard.

## Frontmatter contract (consumed)

The skill reads (but does not write) the MVP spec's frontmatter:

| Field | Used for |
|---|---|
| `spec_type` | Should be `"mvp"`. Skill warns if not. |
| `mode` | Must be `"project"`. Refuse if `"feature"`. |
| `name` | Used as the Railway project name. |
| `template` | Compared against the repo's actual template (Step 2). |
| `status` | Should be `"IMPLEMENTED"` ideally; the build check (Step 1) is the actual gate. |

## Examples

### Example 1 — clean kitchen-sink-ts deploy

User invokes `/railway-deployment` in a freshly-built kitchen-sink-ts repo. Spec at `IMPLEMENTED/IMPLEMENTED_MVP_20260501_SPEC.md` declares `template: kitchen-sink-ts`, `name: PAGEMARK`.

Pre-flight: MCP ✓, web_fetch ✓, on main ✓, clean tree ✓, GitHub remote ✓, `bun run build` succeeds.

Provisioning:
- Railway project `PAGEMARK`, region EU.
- One service `web`, Nixpacks builder, build `bun install && bun run build`, start `bun run start`.
- Postgres provisioned (smallest tier), linked to `web`.
- Dynamic domain `pagemark-web.up.railway.app` provisioned.
- GitHub app already installed → repo linkage via MCP succeeds.

Env vars:
- `SKILL_SET`: `BETTER_AUTH_URL=https://pagemark-web.up.railway.app`, `DATABASE_URL=${{ Postgres.DATABASE_URL }}`, `OPENROUTER_DEFAULT_MODEL=anthropic/claude-haiku-4.5`.
- `USER_SET` (pending): `BETTER_AUTH_SECRET`, `OPENROUTER_API_KEY`. Polar/R2 vars not used in PAGEMARK's build (Polar/R2 marked REMOVE in the MVP spec) — skipped.

Post-deploy command: `bun run db:migrate` on the `web` service.

Deploy: `git push` succeeds (push is no-op since main is already at origin), Railway starts deploy. Build succeeds, post-deploy migration runs, deploy completes.

Health check: `GET https://pagemark-web.up.railway.app/` → 200 OK.

Report at `AGENT_REPORTS/DEPLOY_REPORT_20260501_1612.md`. Two pending USER_SET vars listed for manual entry.

Chat: "Deployed to https://pagemark-web.up.railway.app — 2 env vars need manual entry. Report at AGENT_REPORTS/DEPLOY_REPORT_20260501_1612.md."

### Example 2 — refusal: dirty tree

User invokes with uncommitted changes. Skill refuses: "Working tree has uncommitted changes in `src/lib/foo.ts`. Deploy requires a clean tree on main so the deployed commit matches the spec lifecycle. Commit or stash, then re-invoke."

### Example 3 — refusal: Railway project already exists

User invokes a second time on the same repo. MCP check finds an existing Railway project linked to the GitHub repo. Skill refuses: "This repo is already linked to Railway project `PAGEMARK` (id: `xyz123`). Subsequent deploys are `git push origin main` — Railway picks up automatically. If you want to tear down and re-provision, delete the project in the Railway dashboard first, then re-invoke this skill."

### Example 4 — twotier with two services

User invokes in a fresh twotier repo. `bun run build` builds both server and client workspaces.

Provisioning:
- Project `MYAPP`, region EU.
- Two services: `server` (Hono, start `bun run start` from `server/` workspace), `client` (Vite static build, served via Railway's static hosting).
- Postgres → linked to `server`.
- Two dynamic domains: `myapp-server.up.railway.app`, `myapp-client.up.railway.app`.

Env vars:
- Server SKILL_SET: `PORT=3000`, `NODE_ENV=production`, `BETTER_AUTH_URL=https://myapp-server.up.railway.app`, `TRUSTED_ORIGINS=https://myapp-client.up.railway.app`, `DATABASE_URL=${{ Postgres.DATABASE_URL }}`, `POLAR_SUCCESS_URL=https://myapp-client.up.railway.app/billing/success`, `POLAR_SERVER=production`.
- Client SKILL_SET: any `VITE_*` non-sensitive vars + derived API URL.
- USER_SET: `BETTER_AUTH_SECRET`, `POLAR_ACCESS_TOKEN`, `POLAR_WEBHOOK_SECRET`, `OPENROUTER_API_KEY`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`.

Post-deploy command: `bun run --filter './server' db:migrate` on the `server` service.

Deploy + health check both URLs succeed. Report includes both services.

### Example 5 — statix Dockerfile deploy

User invokes in a fresh statix repo.

Provisioning:
- Project `MYDOCS`, region EU.
- One service `web`, Dockerfile-based (uses repo's `Dockerfile` + `nginx.conf`).
- No Postgres.
- Dynamic domain `mydocs-web.up.railway.app`.
- Service port: 8080 (per `nginx.conf` `listen 8080`).

Env vars: statix has no env, skip configuration entirely.

Post-deploy: none.

Deploy succeeds, health check on `/` → 200. Report written.
