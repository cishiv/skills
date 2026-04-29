# kitchen-sink-twotier — capabilities snapshot

**Snapshot date:** 2026-04-27

Separate Bun + Hono backend and React + Vite frontend, with a `@twotier/shared` workspace for cross-cutting Zod schemas. Three workspaces: `server/`, `client/`, `shared/`. Use this template when the API needs to serve multiple clients, or when there's significant server-side processing that warrants a clean tier separation.

## Stack

- **Runtime / package manager:** Bun. Workspaces.
- **Backend:** Hono on Bun. Controllers + services + routes pattern. Services are the only thing that touches the DB.
- **Frontend:** React + Vite + TanStack Router. Domain UI under `client/src/features/`, pages under `client/src/pages/`, shadcn primitives under `client/src/components/ui/`.
- **Shared:** Zod schemas live in `shared/src/api/`, exported as `@twotier/shared`. The graph is one-directional — `shared/` never imports from `server/` or `client/`.
- **Database:** Postgres via Drizzle ORM. Schema in `server/src/db/schema/*.ts`.
- **Validation:** Zod 4. Server state on the client lives in React Query.

## What's PROVIDED (works out of the box)

- **BetterAuth email/password.** Lazy-init server instance at `server/src/lib/auth.ts`. Catch-all routes at `server/src/routes/auth.routes.ts`. Client at `client/src/lib/auth-client.ts`. Session-cookie based; `users`, `sessions`, `accounts`, `verifications` tables migrated.
- **Auth middleware.** `server/src/middleware/auth.ts`. Apply to route groups; reads cookie via `auth.api.getSession`. Injects `c.var.user`.
- **Drizzle scaffolding.** `bun run --filter './server' db:generate | db:migrate | db:studio | db:doc`.
- **Service / controller / route separation.** Controllers don't touch the DB. Services accept dependencies as arguments (testable). Routes wire middleware and controllers (no logic).
- **Zero-env boot.** The Zod env schema in `server/src/lib/env.ts` is fully optional. App boots with empty env; integrations fail loudly only when invoked via `requireEnv(key)` inside their getter.
- **Lazy-init pattern.** Canonical example: `server/src/lib/storage.ts`.
- **Typed API fetcher.** `client/src/lib/api.ts` for client→server calls. No ad-hoc `fetch()`.
- **Shared Zod schemas.** Every endpoint has its request/response schemas defined in `shared/src/api/*.ts` *before* the controller / hook is written.
- **Errors.** Standard `ApiError` shape in `shared/src/errors.ts` and `server/src/lib/errors.ts`.
- **`auth:generate`.** Regenerates `server/src/db/schema/auth.ts` from BetterAuth config.

## What's CONFIGURABLE (present in scaffold, requires wiring)

- **Polar billing.** Lazy-init SDK at `server/src/lib/polar.ts`. Checkout controller wired. Webhook handler at `server/src/routes/webhooks.routes.ts` validates with `validateEvent(...)`. `subscriptions` table migrated. Wire by setting `POLAR_*` env vars.
- **R2 / S3-compatible storage.** Lazy-init at `server/src/lib/storage.ts`. Presign + complete endpoints scaffolded. Wire by setting `R2_*` (or compatible S3) env vars.
- **OpenRouter LLM calls.** Lazy-init at `server/src/lib/llm.ts`. Use `generateText` / `generateObject` / `streamText` from Vercel AI SDK. Wire by setting `OPENROUTER_API_KEY`.
- **shadcn components.** `cd client && bunx --bun shadcn@latest add <component>`.

## What's NOT PRESENT (must be built / added if needed)

- OAuth providers beyond email/password.
- Email transactional sending (no Resend, Postmark, etc.).
- Background jobs / queues.
- Real-time websockets / SSE.
- Vector / semantic search.
- Multi-tenancy abstractions.
- A monorepo build orchestrator beyond Bun workspaces — Turborepo / Nx are not pre-wired.

## Essential file map

### Server

| Concept | Path |
|---|---|
| Entry | `server/src/index.ts` |
| Env (all optional) | `server/src/lib/env.ts` |
| DB client / schema | `server/src/lib/db.ts`, `server/src/db/schema/*.ts` |
| BetterAuth | `server/src/lib/auth.ts`, `server/src/middleware/auth.ts` |
| Polar / Storage / LLM | `server/src/lib/polar.ts`, `server/src/lib/storage.ts`, `server/src/lib/llm.ts` |
| Services (DB-touching) | `server/src/services/*.service.ts` |
| Controllers (HTTP) | `server/src/controllers/*.controller.ts` |
| Routes (wiring) | `server/src/routes/*.routes.ts` |
| Webhooks | `server/src/routes/webhooks.routes.ts` |
| Drizzle config / migrations | `server/drizzle.config.ts`, `server/src/db/migrations/` |

### Client

| Concept | Path |
|---|---|
| Entry | `client/src/main.tsx` |
| Routes | `client/src/routes/*.tsx` |
| Pages (composition only) | `client/src/pages/*.tsx` |
| Domain features | `client/src/features/<feature>/*` |
| API fetcher / Auth client | `client/src/lib/api.ts`, `client/src/lib/auth-client.ts` |
| shadcn primitives | `client/src/components/ui/*` |

### Shared

| Concept | Path |
|---|---|
| Public Zod schemas | `shared/src/api/*.ts` |
| Error shape | `shared/src/errors.ts` |
| Barrel | `shared/src/index.ts` |

## Commands

| Command | Purpose |
|---|---|
| `bun run dev` | Server on `:3000` and client on `:5173` concurrently |
| `bun run typecheck` | Typecheck every workspace |
| `bun run lint` | Lint every workspace |
| `bun run test` | Test every workspace |
| `bun run --filter './server' db:generate` | Generate migration |
| `bun run --filter './server' db:migrate` | Apply migrations |
| `bun run --filter './server' db:studio` | Drizzle Studio |
| `bun run --filter './server' auth:generate` | Regenerate auth schema from BetterAuth config |
