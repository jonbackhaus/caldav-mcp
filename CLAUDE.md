# Project Instructions for AI Agents

This file provides instructions and context for AI coding agents working on this project.

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:1105d646 -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

**Architecture in one line:** issues live in a local Dolt DB; sync uses `refs/dolt/data` on your git remote; `.beads/issues.jsonl` is a passive export. See https://github.com/gastownhall/beads/blob/main/docs/core-concepts/sync-concepts.md for details and anti-patterns.

## Agent Context Profiles

The managed Beads block is task-tracking guidance, not permission to override repository, user, or orchestrator instructions.

- **Conservative (default)**: Use `bd` for task tracking. Do not run git commits, git pushes, or Dolt remote sync unless explicitly asked. At handoff, report changed files, validation, and suggested next commands.
- **Minimal**: Keep tool instruction files as pointers to `bd prime`; use the same conservative git policy unless active instructions say otherwise.
- **Team-maintainer**: Only when the repository explicitly opts in, agents may close beads, run quality gates, commit, and push as part of session close. A current "do not commit" or "do not push" instruction still wins.

## Session Completion

This protocol applies when ending a Beads implementation workflow. It is subordinate to explicit user, repository, and orchestrator instructions.

1. **File issues for remaining work** - Create beads for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **Handle git/sync by active profile**:
   ```bash
   # Conservative/minimal/default: report status and proposed commands; wait for approval.
   git status

   # Team-maintainer opt-in only, unless current instructions forbid it:
   git pull --rebase
   git push
   git status
   ```
5. **Hand off** - Summarize changes, validation, issue status, and any blocked sync/commit/push step

**Critical rules:**
- Explicit user or orchestrator instructions override this Beads block.
- Do not commit or push without clear authority from the active profile or the current user request.
- If a required sync or push is blocked, stop and report the exact command and error.
<!-- END BEADS INTEGRATION -->


## Build & Test

```bash
pnpm install
pnpm check        # typecheck (src + tests), biome lint, vitest, build — same as CI
pnpm test:watch   # TDD loop
pnpm format       # biome format --write
```

Node >= 22, ESM, TypeScript strict (`noUncheckedIndexedAccess`,
`exactOptionalPropertyTypes`, `verbatimModuleSyntax`). Tests live next to the
code as `src/**/*.test.ts`. Write the failing test first.

## Architecture Overview

A long-running **calendar broker**: an MCP server (Streamable HTTP) that reads
iCloud calendars over CalDAV and serves them to local agents/bots on the LAN.
See `docs/adr/0001-broker-architecture.md` (a draft until its decision beads
are ratified). Core invariants:

- **The broker is the security boundary.** The iCloud app-specific password
  grants full read/write to calendars and contacts and cannot be scoped. Only
  the broker holds it; clients get per-client bearer tokens with scopes.
- **Read-only first.** Write tools come later, gated by a `write` scope, with
  `If-Match` ETags and an audit log. Until then no code path may issue
  PUT/DELETE/PROPPATCH/MKCALENDAR against iCloud.
- **Staleness is visible.** `/health` reports the age of the last successful
  sync; never degrade silently.

## Conventions & Patterns

- **Decision beads** (`type: decision`) are never closed or ratified without
  presenting them to the user and getting an explicit go-ahead.
- **Secrets:** iCloud credentials and client tokens come from 1Password (`op`).
  Never commit, log, or echo them; redact them in error messages. Read a secret
  into a variable and test `[ -n "$VAR" ]` before use: a timed-out `op read`
  yields an empty string with a clean exit.
- **Real calendar data is private.** Spike output goes in `spike/out/`
  (gitignored). Test fixtures must be synthetic or scrubbed `.ics`.
- **Tests never hit iCloud.** Unit tests use synthetic fixtures; live checks
  are opt-in scripts, not part of `pnpm test` or CI.
- ADRs live in `docs/adr/NNNN-<slug>.md`.
