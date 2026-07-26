# l3rain-runner

Public **runner-only** repo for CPDADEV LLC. It contains **no product source and no docs** — just
one GitHub Actions workflow that keeps the l3rain "waker" alive.

## Why it exists (temporary bridge)

The private repo `cpdadev-llc/l3rain` exhausted its monthly pool of **2,000 free private-Actions
minutes**, so its own `waker-hosted.yml` cannot run there until the pool resets. **Public-repo
Actions minutes are unlimited and free** (they are not billed against the private-minute pool), so
this public repo runs the waker on the org's behalf until the private minutes reset.

## How it works

[`.github/workflows/waker-runner.yml`](.github/workflows/waker-runner.yml) mirrors the private
`waker-hosted.yml`. On each `schedule` tick (or manual `workflow_dispatch`) it:

1. Clones the **private** `cpdadev-llc/l3rain` at runtime over HTTPS using a fine-grained PAT
   (`secrets.L3RAIN_PAT`) into a git-ignored `./l3rain` directory.
2. Runs the same `agent-cpd-orchestrator` invocation via Claude Code
   (`secrets.CLAUDE_CODE_OAUTH_TOKEN`) — same no-token guard, resume/refill prompts, usage-sentinel
   burst-on-reset logic, and checkpoint behavior.
3. Commits and pushes breadcrumbs / work products back to `cpdadev-llc/l3rain` with the same PAT.

**The l3rain repo is cloned at runtime and is never committed here.** This repo stays source-free.

## Secrets required (set in Settings → Secrets and variables → Actions)

| Secret | Purpose |
|---|---|
| `L3RAIN_PAT` | Fine-grained PAT with access to `cpdadev-llc/l3rain`: Contents R/W, Actions R/W, Metadata R. Clones l3rain and pushes breadcrumbs back. |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code auth for the orchestrator run. |

Without `L3RAIN_PAT` the run fails fast with a clear error. Without `CLAUDE_CODE_OAUTH_TOKEN` it
writes a "no token — fleet cannot run" breadcrumb and exits cleanly.

## Security

- No `pull_request` trigger — secrets are never exposed to untrusted/fork code.
- `permissions: contents: read` — the workflow needs no write access to this repo.
- Secrets are auto-masked by Actions and never echoed.

## Retirement

Temporary. Once the private minutes reset and the primary waker resumes, disable this workflow or
delete this repo.
