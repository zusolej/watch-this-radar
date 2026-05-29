# Repository Guidelines

This is `whatch-this-radar`, an Astro 6 SSR web app (React 19 islands, Tailwind 4, Supabase auth) deployed to Cloudflare Workers. Product context: `@context/foundation/prd.md`.

## Hard Rules

- API routes (`src/pages/api/**`) must export `const prerender = false` — the app uses `output: "server"`.
- `SUPABASE_URL` and `SUPABASE_KEY` are server-only secrets from `@astro.config.mjs` `env.schema`. Read them via `astro:env/server`; never import into client code.
- New Supabase tables: enable RLS with per-operation, per-role policies. Migrations in `supabase/migrations/` use `YYYYMMDDHHmmss_short_description.sql`.
- No Next.js directives (`"use client"`, etc.) — this is Astro + React islands.
- Merge Tailwind classes with `cn()` from `@/lib/utils`; don't concatenate strings manually.
- Recurring research jobs run as a separate execution path, not inside a request handler (see `@context/foundation/tech-stack.md`).

## Project Structure

- `src/pages/` — Astro pages and `api/` endpoints (file-based routing).
- `src/components/` — Astro for static/layout, React (`.tsx`) only when interactive. shadcn "new-york" primitives live in `src/components/ui/`.
- `src/components/hooks/` — extracted React hooks.
- `src/lib/` — services and helpers; business logic under `src/lib/services/`.
- `src/middleware.ts` — request middleware; gate routes via the `PROTECTED_ROUTES` array.
- `src/types.ts` — shared entities and DTOs.
- `supabase/` — local Supabase config and `migrations/`.
- `context/` — PRD, stack hand-off, bootstrap log. Start with `@context/foundation/prd.md`.

## Build, Run, and Deploy Commands

- `npm run dev` — Astro dev server on Cloudflare workerd.
- `npm run build` — SSR build via `@astrojs/cloudflare`.
- `npm run preview` — preview the production build.
- `npm run lint` / `npm run lint:fix` — ESLint with type-checked rules; CI runs `lint`.
- `npm run format` — Prettier across the repo.
- `npx supabase start` / `stop` — local Supabase (requires Docker).
- `npx wrangler deploy` — deploy to Cloudflare Workers.

## Coding Style & Naming

- TypeScript strict (`@tsconfig/strict`) plus `strictTypeChecked` + `stylisticTypeChecked` (see `@eslint.config.js`).
- Prettier: 2-space indent, double quotes, semicolons, `printWidth: 120`, `trailingComma: "all"` (`@.prettierrc.json`).
- Path alias `@/*` → `./src/*` (`@tsconfig.json`).
- API handlers export uppercase `GET`, `POST`; validate input with zod.
- Install shadcn primitives with `npx shadcn@latest add <name>`.
- `no-console` is `warn` — drop console statements before commit.

## Commits, PRs, and CI

- Commit convention is still being defined (history is sparse). Use a short imperative subject line.
- CI (`@.github/workflows/ci.yml`) on push/PR to `master` runs `npm ci`, `npx astro sync`, `npm run lint`, `npm run build`; build needs `SUPABASE_URL` and `SUPABASE_KEY` repo secrets.
- Pre-commit hook (husky + lint-staged) runs `eslint --fix` on `*.{ts,tsx,astro}` and `prettier --write` on `*.{json,css,md}`. Do not skip it.

## Security & Configuration

- Node 22.14.0 (`@.nvmrc`).
- Local env: copy `.env.example` to `.env` (Node) and `.dev.vars` (Cloudflare local). Both are gitignored.
- Production secrets via `npx wrangler secret put` or the Cloudflare dashboard.
