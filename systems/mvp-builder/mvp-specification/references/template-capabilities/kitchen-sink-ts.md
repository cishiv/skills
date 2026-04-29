# kitchen-sink-ts — capabilities snapshot

**Snapshot date:** 2026-04-27

Single Bun process running TanStack Start (React 19, Vite 7, SSR). One deployable. File-based routing.

## Stack

- **Runtime / package manager:** Bun.
- **Framework:** TanStack Start (React 19, Vite 7) with file-based routing under `src/routes/`. Path alias `@/*` → `src/*`.
- **Database:** Postgres via Drizzle ORM. Schema in `src/lib/db/schema.ts`. Migrations in `drizzle/`.
- **UI:** shadcn/ui (new-york, neutral) on Tailwind 4. shadcn primitives in `src/components/ui/`.
- **Validation:** Zod 4. `verbatimModuleSyntax`. Type-only imports for types.

## What's PROVIDED (works out of the box)

- **BetterAuth email/password.** Server instance in `src/lib/auth.ts`, client in `src/lib/auth-client.ts`. Catch-all handler at `src/routes/api/auth/$.ts`. Session-cookie based. `users`, `sessions`, `accounts`, `verifications` tables already migrated.
- **Auth-gated routes.** `src/routes/_protected.tsx` calls `getServerAuthUser` in `beforeLoad` and redirects unauthenticated users. Place protected routes under `src/routes/_protected/`.
- **Auth-gated API routes.** `authMiddleware` at `src/middleware/auth-middleware.ts`. Apply in route handlers; read `context.user` and `context.session`.
- **Drizzle scaffolding.** Migration generation, apply, studio, mermaid schema dump (`bun run db:generate`, `db:migrate`, `db:studio`, `db:doc`).
- **Adapter pattern.** Domain types and DB→public adapters in `src/lib/adapters/`. Public types in `src/lib/types.ts`.
- **SSR + hydration.** Server Components by default. `'use client'` for interactivity.
- **SEO helper.** `src/lib/seo.ts` produces meta-tag arrays for `head()`.
- **Blog scaffolding.** Markdown blog at `src/content/blog/*.md`, loader in `src/lib/blog.ts`.
- **Lazy-init pattern.** All env-dependent SDKs are lazy-initialized; the app boots with empty `.env`. Canonical example: `src/lib/r2.ts`.
- **Railway deploy config.** `railway.json` ready.

## What's CONFIGURABLE (present in scaffold, requires wiring)

- **Polar billing.** Lazy-init SDK at `src/lib/polar.ts`. Webhook handler at `src/routes/api/webhooks/polar.ts`. `subscriptions` table migrated. Wire by setting `POLAR_*` env vars and configuring products in Polar dashboard.
- **Cloudflare R2 file storage.** Lazy-init at `src/lib/r2.ts`. Upload endpoints under `src/routes/api/uploads/*`. `uploads` table migrated. Wire by setting `R2_*` env vars and creating a bucket.
- **OpenRouter LLM calls.** Lazy-init at `src/lib/openrouter.ts`. Completion endpoint at `src/routes/api/ai/complete.ts`. Use `complete()` for text or `completeStructured()` for schema-constrained output. Wire by setting `OPENROUTER_API_KEY` and optionally `OPENROUTER_DEFAULT_MODEL`.
- **shadcn components beyond what's pre-installed.** `bunx --bun shadcn@latest add <component>` installs into `src/components/ui/`.

## What's NOT PRESENT (must be built / added if needed)

- OAuth providers beyond email/password (Google, GitHub, etc.).
- Email transactional sending (no Resend, Postmark, etc. wired).
- Background jobs / queues.
- Real-time websockets / SSE.
- Vector / semantic search (no pgvector, no Pinecone wired).
- Multi-tenancy abstractions.
- Multi-region / sharding.
- A separate API tier — this is single-process. Use `kitchen-sink-twotier` if a separate API is required.

## Essential file map

| Concept | Path |
|---|---|
| Routes | `src/routes/` |
| API routes | `src/routes/api/` |
| Auth (server / client / middleware) | `src/lib/auth.ts`, `src/lib/auth-client.ts`, `src/middleware/auth-middleware.ts` |
| DB client / schema | `src/lib/db/index.ts`, `src/lib/db/schema.ts` |
| Adapters | `src/lib/adapters/*.adapter.ts` |
| Public types | `src/lib/types.ts` |
| SEO helper | `src/lib/seo.ts` |
| Drizzle config | `drizzle.config.ts` |
| Migrations | `drizzle/` |
| Database model doc | `DATABASEMODEL.md` |
| Spec workflow | `SPECIFICATIONS/HOW_TO_USE_SPECIFICATION.md` |

## Commands

| Command | Purpose |
|---|---|
| `bun run dev` | Vite dev server on port 3000 |
| `bun run build` | Production build → `dist/server/server.js` |
| `bun run start` | `bun dist/server/server.js` |
| `bun run test` | Vitest |
| `bun run check` | Prettier write + ESLint --fix |
| `bun run db:generate` | Generate migration from schema |
| `bun run db:migrate` | Apply migrations |
| `bun run db:studio` | Drizzle Studio |
