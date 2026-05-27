---
bootstrapped_at: 2026-05-25T22:15:44Z
starter_id: 10x-astro-starter
starter_name: 10x Astro Starter (Astro + Supabase + Cloudflare)
project_name: whatch-this-radar
language_family: js
package_manager: npm
cwd_strategy: git-clone
bootstrapper_confidence: first-class
phase_3_status: ok
audit_command: npm audit --json
---

## Hand-off

```yaml
starter_id: 10x-astro-starter
package_manager: npm
project_name: whatch-this-radar
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-pages
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: first-class
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: true
  has_payments: false
  has_realtime: false
  has_ai: false
  has_background_jobs: true
```

A solo, after-hours web-app MVP with a two-week timeline benefits from an opinionated JavaScript starter that gives auth, data, and deployment defaults quickly instead of spending time assembling infrastructure by hand. `10x-astro-starter` is the recommended default for this project shape, and `cloudflare-pages` plus `github-actions` keeps the path to first deploy short. The main trade-off is recurring research: it should run through a separate background execution path rather than as a long-running web request in the app runtime, while the app focuses on account management and showing completed reports.

## Pre-scaffold verification

| Signal      | Value   | Severity | Notes |
| ----------- | ------- | -------- | ----- |
| npm package | not run | not run  | Starter command begins with `git clone`, so no `create-*` npm package recency check applied. |
| GitHub repo | not run | not run  | `gh` CLI is unavailable locally, so repo recency could not be queried. |

## Scaffold log

**Resolved invocation**: `git clone https://github.com/przeprogramowani/10x-astro-starter .bootstrap-scaffold && cd .bootstrap-scaffold && npm install`  
**Strategy**: git-clone  
**Exit code**: 0  
**Files moved**: 20  
**Conflicts (.scaffold siblings)**: none  
**.gitignore handling**: moved silently  
**.bootstrap-scaffold cleanup**: deleted

## Post-scaffold audit

**Tool**: `npm audit --json`  
**Summary**: 0 CRITICAL, 1 HIGH, 9 MODERATE, 0 LOW  
**Direct vs transitive**: 0/0/2/0 direct of total 0/1/9/0

#### CRITICAL findings

None.

#### HIGH findings

- **devalue** — transitive dependency. Advisory: `GHSA-77vg-94rm-hx3p`. Issue: DoS via sparse array deserialization. Fix available: yes.

#### MODERATE findings

- **@astrojs/check** — direct dependency. Advisory chain via `@astrojs/language-server`. Fix available: `@astrojs/check@0.9.2` (semver-major).
- **@astrojs/language-server** — transitive dependency. Advisory chain via `volar-service-yaml`. Fix available through `@astrojs/check@0.9.2`.
- **@cloudflare/vite-plugin** — transitive dependency. Advisory chain via `miniflare`, `wrangler`, and `ws`. Fix available: yes.
- **miniflare** — transitive dependency. Advisory chain via `ws`. Fix available: yes.
- **volar-service-yaml** — transitive dependency. Advisory chain via `yaml-language-server`. Fix available through `@astrojs/check@0.9.2`.
- **wrangler** — direct dependency. Advisory chain via `miniflare`. Fix available: yes.
- **ws** — transitive dependency. Advisory: `GHSA-58qx-3vcg-4xpx`. Issue: uninitialized memory disclosure. Fix available: yes.
- **yaml** — transitive dependency. Advisory: `GHSA-48c2-rrv3-qjmp`. Issue: stack overflow via deeply nested YAML collections. Fix available through `@astrojs/check@0.9.2`.
- **yaml-language-server** — transitive dependency. Advisory chain via `yaml`. Fix available through `@astrojs/check@0.9.2`.

#### LOW / INFO findings

None.

## Hints recorded but not acted on

| Hint | Value |
| ---- | ----- |
| bootstrapper_confidence | first-class |
| quality_override | false |
| path_taken | standard |
| self_check_answers | null |
| team_size | solo |
| deployment_target | cloudflare-pages |
| ci_provider | github-actions |
| ci_default_flow | auto-deploy-on-merge |
| has_auth | true |
| has_payments | false |
| has_realtime | false |
| has_ai | false |
| has_background_jobs | true |

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:
- `git init` (if you have not already) to start your own repo history.
- Review any `.scaffold` siblings the conflict policy created and decide which version of each file to keep.
- Address audit findings per your project's risk tolerance — the full breakdown is in this log.
