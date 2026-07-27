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
2. **Collects its own exhaust:** archives every prior completed run's log archive into the private
   repo, then deletes that run from this public repo (see *Self-cleaning* below).
3. Runs the same `agent-cpd-orchestrator` invocation via Claude Code
   (`secrets.CLAUDE_CODE_OAUTH_TOKEN`) — same no-token guard, usage-sentinel burst-on-reset logic,
   and checkpoint behavior. The orchestrator's prompts are **not stored here**; they are read from
   the private clone at runtime, with a generic fallback in the workflow.
4. Commits and pushes breadcrumbs / work products back to `cpdadev-llc/l3rain` with the same PAT.

**The l3rain repo is cloned at runtime and is never committed here.** This repo stays source-free.

## Self-cleaning

Public-repo Actions logs never expire, so an untended bridge would accumulate a permanent,
world-readable record of every tick. Each run therefore begins by collecting the ones before it:

- list this workflow's **completed** runs (excluding the run doing the collecting);
- download each one's log archive and commit it into the private repo as
  `run-<id>-logs.zip` (already-archived ids are never re-downloaded);
- push — and **only** once the archive is confirmed on the private remote, `DELETE` that run
  from this repo.

Archive-then-delete, never the reverse: if the push fails, nothing is deleted and the next tick
retries. The step is fail-soft — log hygiene never blocks the fleet — and runs before the guards
so it also collects on halted, no-token and dry-run ticks. A run cannot delete itself, so the most
recent completed run stays visible until the next tick collects it; that is the floor.

## Secrets required (set in Settings → Secrets and variables → Actions)

| Secret | Purpose |
|---|---|
| `L3RAIN_PAT` | Fine-grained PAT with access to `cpdadev-llc/l3rain`: Contents R/W, Actions R/W, Metadata R. Clones l3rain and pushes breadcrumbs back. |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code auth for the orchestrator run. |

Without `L3RAIN_PAT` the run fails fast with a clear error. Without `CLAUDE_CODE_OAUTH_TOKEN` it
writes a "no token — fleet cannot run" breadcrumb and exits cleanly.

## Security

- No `pull_request` trigger — secrets are never exposed to untrusted/fork code.
- `permissions: contents: read` + `actions: write`. The write scope exists for exactly one
  purpose: deleting this repo's own already-archived runs. It grants no access to the repo's
  contents and none at all to the private repo.
- Secrets are auto-masked by Actions and never echoed.
- Nothing from the private repo is printed: no revisions, no branch names, no orchestrator
  transcript, and no prompt text. There are no artifacts — the failed-run transcript is stashed
  inside the private repo instead.

## Retirement

Temporary. Once the private minutes reset and the primary waker resumes, disable this workflow or
delete this repo.
