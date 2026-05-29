---
project: whatch-this-radar
researched_at: 2026-05-29
recommended_platform: Cloudflare Workers + Pages
runner_up: Vercel
context_type: mvp
tech_stack:
  language: TypeScript
  framework: Astro 6 SSR
  runtime: Cloudflare Workers (workerd / V8 isolates)
---

## Recommendation

**Deploy on Cloudflare Workers + Pages.**

The tech stack (`@astrojs/cloudflare` adapter, `workerd` runtime, `wrangler` tooling) is already hard-wired for Cloudflare — zero adapter migration, near-perfect dev/prod parity via `astro dev` running on the real `workerd` runtime since Astro 6. The free tier (100k req/day) covers the MVP traffic envelope at $0, the user has prior Cloudflare experience (interview Q3), and the platform scored 5/5 on all agent-friendly criteria. The primary trade-off — background job execution (recurring research) — requires either the Paid plan ($5/month) for Durable Objects or an external trigger; this risk is documented in the risk register and was acknowledged by the user.

## Platform Comparison

| Platform | CLI-first | Managed/Serverless | Agent-readable docs | Stable deploy API | MCP / Integration | Total |
|---|---|---|---|---|---|---|
| **Cloudflare** | Pass | Pass | Pass | Pass | Pass | **5/5** |
| **Vercel** | Pass | Pass | Pass | Pass | Pass | **5/5** |
| **Netlify** | Partial | Pass | Pass | Pass | Pass | **4.5/5** |
| Railway | Partial | Pass | Pass | Partial | Pass | **4/5** |
| Render | Partial | Pass | Partial | Partial | Pass | **3.5/5** |
| Fly.io | Partial | Partial | Partial | Pass | Fail | **2.5/5** |

Soft-weight adjustments applied: cost-sensitivity (Q2) penalized Fly.io (no free tier, $6–10/mo) and Railway ($5/mo mandatory); existing Cloudflare experience (Q3) broke the tie between Cloudflare and Vercel; single-region preference (Q4) was neutral; external providers OK (Q5) removed any co-location advantage for Fly.io/Railway.

### Shortlisted Platforms

#### 1. Cloudflare Workers + Pages (Recommended)

The current codebase uses `@astrojs/cloudflare` and `wrangler` — no adapter swap required. `wrangler deploy` / `wrangler rollback` / `wrangler tail` form a complete CLI-first operational loop. Docs are fully LLM-accessible (`/llms.txt`, per-product `llms-full.txt`, per-page `.md`). A first-party MCP server (GA) covers deploy, Workers, observability. Free tier covers 100k req/day at 10 ms CPU/invocation. The recurring research background job is the one area that needs external coordination — addressed in the risk register.

#### 2. Vercel

Scored 5/5 on criteria, `@astrojs/vercel` adapter is GA for Astro 6, and the Hobby free tier ($0) comfortably handles 10k–100k req/month. Fluid Compute (GA since April 2025, default for new projects) eliminates cold starts. Vercel MCP is GA with OAuth 2.1. The gap vs. Cloudflare: adapter swap required (`@astrojs/cloudflare` → `@astrojs/vercel`), and `process.env` replaces `astro:env/server` Workers bindings — meaningful refactor for an already-bootstrapped project.

#### 3. Netlify

Scored 4.5/5 — rollback is dashboard-only (no CLI command), which is the only partial. Free 300-credit plan covers MVP traffic at $0. `@astrojs/netlify` adapter is GA and day-one Astro 6 compatible. Official MCP server (`@netlify/mcp`) is GA with some auth friction reported. Requires adapter swap from `@astrojs/cloudflare`. Dropped below Vercel due to no-CLI-rollback and adapter migration cost.

## Anti-Bias Cross-Check: Cloudflare Workers + Pages

### Devil's Advocate — Weaknesses

1. **CPU limit 10 ms/invocation (free tier)** — Supabase I/O is excluded (network wait), but React SSR rendering + shadcn + Tailwind template compilation can consume 5–8 ms of pure CPU per complex page. Any page near that ceiling has no safety margin. Workaround: upgrade to Paid ($5/month) for 30 ms CPU cap.
2. **Recurring research background job requires Paid plan or external trigger** — Workers Cron Triggers on the free tier share the 10 ms CPU cap. An LLM + Brave Search pipeline takes 30–90 s wall-clock; the in-Worker processing (prompt assembly, response parsing) needs CPU headroom. Durable Objects (needed for job state tracking) are Paid-only. MVP options: (a) upgrade to $5/month Paid + use Cloudflare Workflows/Queues, (b) use a GitHub Actions `schedule:` cron to call a protected API endpoint that triggers the research.
3. **Multi-environment deploy requires a build-time workaround** — `wrangler deploy --env staging` does not work with Astro 6 (issue #15917). CI must run `CLOUDFLARE_ENV=staging npx astro build && wrangler deploy` — two separate steps, not a single command.
4. **Active bugs in `@astrojs/cloudflare` v13** — issue #15946 (`prerenderEnvironment: 'node'` + server islands conflict) and #16456 (hybrid routing 500s). Both are avoided by `output: "server"` (already configured in this project), but upstream adapter upgrades can reintroduce edge cases.
5. **Codebase is deeply tied to Workers runtime** — `cloudflare:workers` imports, `astro:env/server` bindings, `nodejs_compat` flag, `.dev.vars`. Migrating to any other platform later is a full adapter swap + integration retest, not a config change.

### Pre-Mortem — How This Could Fail

The team deployed `whatch-this-radar` on Cloudflare Pages. The Astro SSR frontend worked well from day one — auth, report display, all fast and free. The failure emerged during background job implementation. First attempt: a Cron Trigger Worker calling GPT-4o-mini + Brave Search. In dev: fine. In production on the free tier: the Worker hit the 10 ms CPU limit during response parsing after the LLM returned. Upgrading to Paid ($5/month) fixed the CPU limit, but stateful job coordination (which users have active research, which runs are in-flight) required Durable Objects — another Paid feature with a different programming model. The developer spent 3–4 days learning the Durable Objects API, debugging race conditions in the `workerd` environment where standard Node.js debugging tools don't apply. Separately, the Astro adapter had a regression in a minor version that caused 500s on a hybrid SSR/static page — diagnosing it through `wrangler tail` logs was harder than expected because stack traces in V8 isolates are less descriptive than in Node.js. The MVP shipped, but 2× over the estimated timeline for the background job subsystem, and the final cost was $5/month — the same as any Node.js PaaS — with more operational complexity.

### Unknown Unknowns

1. **Workers Cron Triggers are a separate Worker script** — not bundled with the Pages deployment. Recurring research jobs need a separate `wrangler.toml` / project, separate logs, and a separate deploy step in CI.
2. **`nodejs_compat` + specific npm packages fail silently** — packages using `node:crypto` streams, `node:events` non-standard patterns, or `node:buffer` edge cases may throw only in production Workers runtime, even though `astro dev` (workerd) catches most issues since Astro 6.
3. **`@supabase/supabase-js` edge behavior** — the Workers-compatible Supabase client variant handles auth and REST correctly, but some admin API methods and storage operations behave differently than in Node.js. Test auth flows specifically in `wrangler dev` before relying on them in production.
4. **Free tier 10 ms CPU does not accumulate** — there is no "unused CPU credit" that carries over. Each request gets exactly 10 ms regardless of traffic patterns or time of day.
5. **Custom response headers require Workers middleware, not `_headers` file** — Content-Security-Policy, HSTS, and other security headers must be set in Astro middleware (`src/middleware.ts`), not via a Cloudflare Pages `_headers` file. Mixing the two causes unpredictable header precedence in production.

## Operational Story

- **Preview deploys**: `wrangler deploy --env preview` (or GitHub Actions `branch-preview` job) publishes a separate Workers deployment. Preview URLs are not password-protected by default — enable Cloudflare Access (zero-trust) on the preview subdomain if the app handles real user data during staging.
- **Secrets**: Workers Secrets set via `npx wrangler secret put SUPABASE_URL` (per-environment). Also available as Cloudflare Dashboard env vars under Workers & Pages → Settings → Environment Variables. Never commit to `.dev.vars`. In CI, pass secrets as GitHub Actions secrets injected into `wrangler deploy` via env.
- **Rollback**: `wrangler rollback` reverts to the previous production deployment in seconds. For a specific version: `wrangler versions list` then `wrangler rollback <VERSION_ID>`. Note: DB migrations (Supabase) do not auto-rollback — plan rollback-safe migration strategies.
- **Approval**: Agents may run `wrangler deploy`, `wrangler tail`, `wrangler versions list`, and `wrangler rollback` unattended in CI. Rotating `SUPABASE_KEY` or changing Cloudflare account-level settings requires human approval in the Cloudflare dashboard.
- **Logs**: `wrangler tail` streams live request logs to the terminal. Filter: `wrangler tail --status error --format json`. For historical logs: Cloudflare Workers Logs in the dashboard (retained 7 days on free tier). No MCP log-reading tool exists yet — use CLI.

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| Free tier 10 ms CPU cap hit by complex SSR pages | Devil's advocate | M | M | Profile CPU with `wrangler dev --inspect` before launch; upgrade to Paid ($5/mo) if any route exceeds 8 ms |
| Recurring research job requires Paid plan or external trigger | Devil's advocate | H | H | For MVP: use GitHub Actions `schedule:` cron calling a protected `/api/run-research` endpoint; avoids Durable Objects until needed |
| Multi-env deploy workaround breaks CI | Devil's advocate | L | M | Document `CLOUDFLARE_ENV=staging npx astro build && wrangler deploy` in CI YML; test in GitHub Actions before shipping |
| `@astrojs/cloudflare` adapter regression on upgrade | Devil's advocate | M | M | Pin adapter version in `package.json`; only upgrade with explicit changelog review |
| Vendor lock-in makes future platform migration costly | Devil's advocate | L | M | Keep business logic in `src/lib/services/` decoupled from Cloudflare-specific imports; only `src/pages/api/` and middleware touch platform bindings |
| LLM + search pipeline exceeds Workers CPU in background job | Pre-mortem | H | H | Implement background research outside Workers runtime (GitHub Actions cron or external queue) for MVP; migrate to Cloudflare Workflows post-validation |
| Durable Objects complexity underestimated | Pre-mortem | M | M | Avoid Durable Objects for MVP; use simple DB-backed job state in Supabase instead |
| `@supabase/supabase-js` auth methods behave differently on Workers | Unknown unknowns | M | H | Run full auth flow integration test via `wrangler dev` before first production deploy |
| CSP / security headers misconfigured via `_headers` file | Unknown unknowns | M | M | Set all response headers in `src/middleware.ts`; remove any `_headers` file from the project |
| Cron Trigger deployed as separate script creates operational blind spot | Unknown unknowns | L | M | Add Cron Trigger Worker to the same GitHub Actions workflow and CI checks as the main app |

## Getting Started

1. **Verify `wrangler` version** — the project uses Wrangler 4.x: `npx wrangler --version`. Upgrade if behind: `npm install wrangler@latest --save-dev`.
2. **Authenticate** — `npx wrangler login` opens a browser OAuth flow. Credentials are stored in `~/.config/.wrangler/`.
3. **Local dev with real Workers runtime** — `npm run dev` already runs `astro dev` backed by `workerd` (Astro 6 + `@astrojs/cloudflare` v13). No separate `wrangler dev` needed.
4. **First deploy** — `npm run build && npx wrangler deploy`. The `wrangler.jsonc` `assets.directory` must point to `./dist`. Check that `SUPABASE_URL` and `SUPABASE_KEY` are set as Workers Secrets: `npx wrangler secret put SUPABASE_URL`.
5. **Verify rollback works** — after first successful production deploy, run `npx wrangler versions list` to confirm the version is recorded. Test `npx wrangler rollback` in a staging environment before needing it in anger.

## Out of Scope

The following were not evaluated in this research:
- Docker image configuration
- CI/CD pipeline setup (GitHub Actions workflow details)
- Production-scale architecture (multi-region, HA, DR)
