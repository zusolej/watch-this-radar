---
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
---

## Why this stack

A solo, after-hours web-app MVP with a two-week timeline benefits from an opinionated JavaScript starter that gives auth, data, and deployment defaults quickly instead of spending time assembling infrastructure by hand. `10x-astro-starter` is the recommended default for this project shape, and `cloudflare-pages` plus `github-actions` keeps the path to first deploy short. The main trade-off is recurring research: it should run through a separate background execution path rather than as a long-running web request in the app runtime, while the app focuses on account management and showing completed reports.
