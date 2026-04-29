# statix — capabilities snapshot

**Snapshot date:** 2026-04-27

Static markdown site generator. Preact + Vite + TypeScript. Client-side SPA. Content baked at build time as `src/content.json` / `src/graph.json`. No SSR. No server. No database. No env. Output is static files served by nginx (`Dockerfile` + `nginx.conf`).

Use this template for documentation sites, blogs, internal wikis, or any read-only content site with wikilink-style cross-references.

## Stack

- **Runtime / package manager:** Bun for build scripts (`bun --bun vite ...`). Vitest runs under Node (the worker model in vitest 2 misbehaves under Bun).
- **Framework:** Preact + Vite + TypeScript. Client-side SPA.
- **Content pipeline:** Markdown under `docs/` → `scripts/build-content.ts` produces `src/content.json` and `src/graph.json` at build time. Wikilinks (`[[Page Name]]`) resolved by `scripts/wikilink-resolver.ts`. Routes and sidebar derived from the file tree — no hand-written route table.
- **Styles:** All in `src/styles.css`. CSS custom properties for customization. Light/dark via `prefers-color-scheme`. Respects `prefers-reduced-motion`.
- **Deploy:** `Dockerfile` builds and serves via nginx (`nginx.conf`).

## What's PROVIDED (works out of the box)

- **Markdown build pipeline.** Drop a `.md` under `docs/`, the directory layout becomes the URL path. `docs/guides/intro.md` → `/guides/intro`.
- **Frontmatter parsing.** Reads `title`, `summary`, optional `date`, optional `tags` from each markdown file. Frontmatter shape documented in `README.md`.
- **Wikilinks.** `[[Other Page]]` resolved at build time. Broken links surface as warnings, do not fail the build.
- **Sidebar / route generation.** Derived from the file tree. No hand-maintained config.
- **Light/dark mode.** Follows OS preference; respects reduced motion.
- **Static deploy via nginx.** `Dockerfile` and `nginx.conf` ready.
- **Vitest.** Build pipeline and wikilink resolver have unit tests in `scripts/*.test.ts`.

## What's CONFIGURABLE (present in scaffold, requires only content)

- **Content.** Author markdown under `docs/`. The pipeline takes care of the rest.
- **Theme tweaks.** `src/styles.css` with CSS custom properties.

## What's NOT PRESENT (must be built / added if needed — and most should not be in statix)

- **No backend.** No API, no auth, no DB, no env. If any of these are needed, statix is the wrong template — consider `kitchen-sink-ts` or `kitchen-sink-twotier`.
- **No search.** No client-side or server-side search index.
- **No authentication.** Content is fully public.
- **No comments / dynamic content.** Static site only.
- **No SSR.** SPA with client-side routing.
- **No CMS integration.** Content is filesystem-only.

## Essential file map

| Concept | Path |
|---|---|
| Build pipeline | `scripts/build-content.ts` |
| Wikilink markdown plugin | `scripts/markdown-wikilink.ts` |
| Wikilink resolver | `scripts/wikilink-resolver.ts` |
| Root component | `src/app.tsx` |
| Components | `src/components/*.tsx` |
| Styles | `src/styles.css` |
| Shared types | `src/types.ts` |
| Markdown content | `docs/` |
| Generated content (gitignored) | `src/content.json`, `public/_docs/` |
| Vite config + plugins | `vite.config.ts` |
| Deploy | `Dockerfile`, `nginx.conf` |
| Spec workflow | `SPECIFICATIONS/HOW_TO_USE_SPECIFICATION.md` |

## Commands

| Command | Purpose |
|---|---|
| `bun run dev` | Vite dev server with hot reload |
| `bun run build` | Build static site to `dist/` |
| `bun run preview` | Preview the built site locally |
| `bun run test` | Vitest (run once) |
| `bun run test:watch` | Vitest (watch mode) |

## Notes for `/mvp-specification`

- The `Integrations` section is **omitted** in statix MVP specs (there are no scaffold integrations to KEEP/REMOVE/NOT_PRESENT against).
- Replace `Data model changes` with `Content model changes` (frontmatter fields, sidebar grouping, page taxonomy).
- Acceptance criteria for visual / typographic concerns will frequently carry `[USER_VERIFIES]` since DOM assertions only cover so much for content sites.
