# mtree — Epic & Ticket Breakdown

> Decomposition of [PRD.md](PRD.md) into a buildable backlog, organized by the 4-phase rebuild plan. Reconstructed *as if greenfield* — each ticket maps to a P-requirement in the PRD.
>
> **Tags:** `[CORE]` repo-agnostic engine · `[TALENTA]` talenta-core-specific · `[UX]` ergonomics/interactive · `[AUTO]` automation/agents · `[DOCS]` · `[QA]`
> **Estimates** are relative story points (Fibonacci). Total ≈ **105 SP** (Phases 1–3.5 core scope; P2/future excluded ≈ **94 SP**).

---

## EPIC — mtree: one-command git-worktree concurrency for talenta-core

**Goal:** collapse a context switch to one command; make parallel work safe by default; keep it a single auditable file.
**Definition of done:** Phases 1–3 shipped, `init` onboards a new engineer in < 2 min, `doctor` reports clean on a freshly provisioned lane, `--json` consumed by at least one automation.
**Success metrics:** see PRD § Success Metrics.

---

## Phase 1 — Core engine (repo-agnostic) · ~34 SP

### MT-1 · `[CORE]` Repo discovery + layered config resolution — **5**
Resolve `MAIN` from `git worktree list --porcelain` (works from any worktree); resolve every tunable `git config wt.<key>` → env → inferred default; base = `wt.base` → `$TCW_BASE` → `origin/HEAD` → `develop/main/master` → current branch. *(PRD P0-4)*
- **AC** Given any worktree of a repo, when mtree runs, then `MAIN`/`PARENT`/`REPO` resolve correctly.
- **AC** `git config wt.base develop` pins the base; absent config falls through the chain.
- **AC** Outside a git repo → hard error.

### MT-2 · `[CORE]` Ticket-type registry + dispatch fallthrough — **5**
`type|prefix|web-base|mobile-base|mode` table; unknown first positional treated as a possible type so a row enables both `work <type>` and bare `<type>`. *(PRD P0-2)*
- **AC** Built-ins present: `bugfix, subtask, fasttrack, task, story, improvement, techdebt(chore), feature, epic, spike`.
- **AC** `types` prints the registry; omitted type → `feature`; `epic` → `integration` mode.
- **AC** Adding one row makes the new type work both ways with no other code change.

### MT-3 · `[CORE]` Lane provisioning engine (`work` / `<type>`) — **8**
One engine creates `<prefix>/<TICKET>[-desc]` off the resolved base (or reuses an existing branch); if already checked out, points there instead of failing. *(PRD P0-1)*
- **AC** Lane created at `<parent>/<repo>-worktrees/<repo>-<slug>` on the right branch, then bootstrapped.
- **AC** Existing-branch / already-checked-out cases are idempotent (no cryptic git error).
- **AC** Non-Jira-looking key → warn but continue.

### MT-4 · `[CORE]` Dependency bootstrap — clone-or-install, never symlink — **8**
Per dep dir: lockfile-hash matches main → APFS COW clone (`cp -c -R`); else real install. Copy `.env` only. Auto-select stack by manifest (`go.mod` → Go-only; PHP → composer+yarn). *(PRD P0-3)*
- **AC** Matching lockfile + clonefile → reflink; clonefile unavailable → fall back to install.
- **AC** Symlinked dep dirs never created; `--no-bootstrap` skips provisioning.
- **AC** Go branch is exclusive (no composer/yarn even if `yarn.lock` exists).

### MT-5 · `[CORE]` `remove` + safe teardown — **5**
Drop a lane, keep the local branch, never touch remotes; block on uncommitted changes (recovery guidance) unless `--force`; `--force` also unlocks locked lanes; never blind `rm -rf`. *(PRD P0-5)*
- **AC** Dirty lane blocked with recovery options; `--force` discards.
- **AC** Refuses to remove `main` / persistent `mobile`; ambiguous ticket match refuses + lists.
- **AC** Orphaned (manually deleted) lane → prune stale metadata only.

### MT-6 · `[CORE]` `cleanup` — merged-lane GC — **5**
Remove only clean **and** merged lanes (ancestry **or** `git cherry` content-equivalence for squash/rebase); skip main/mobile/locked/dirty/detached with reasons; `rmdir` empty container. *(PRD P0-5)*
- **AC** Squash-merged branch is detected as merged and removed.
- **AC** Each skip prints its reason; branch deleted only if fully merged.

### MT-7 · `[CORE]` `ls` / `doctor` / `path` — inspection & health — **5**
Width-aware `ls` table; `doctor` flags symlinked deps, `.env` key drift, prunable/locked; `path` prints a lane's absolute path. *(PRD P0-6)*
- **AC** `ls` never overflows the terminal (real width via `stty`).
- **AC** `doctor` returns non-clean when a symlinked dep or `.env` drift exists.

---

## Phase 2 — talenta-core specialization · ~21 SP

### MT-8 · `[TALENTA]` Idempotent `init` + persistent `mobile` lane — **5**
Validate origin (hard-fail non-talenta unless `--force`/`TCW_ALLOW_ANY_REPO`); create `.worktreeinclude`; install shell-aware `mtree`+`mwt` aliases; provision Mobile-API lane. *(PRD P0-7)*
- **AC** Re-running `init` is a no-op ("No action required").
- **AC** Aliases installed once, shell-aware (zsh/bash).

### MT-9 · `[TALENTA]` Web/Mobile platform split — **3**
`--web` (default)/`--mobile` selects the base column; `--base` overrides; `hotfix [web|mobile] <TICKET>` defaults to web off prod base. *(PRD P0-8)*
- **AC** `--mobile task TM-1` bases off `master-mobile-api`; `--base` always wins.
- **AC** `hotfix TM-1` == `hotfix web TM-1`; stray extra arg → clear error.

### MT-10 · `[TALENTA]` composer-in-laradock + relativized worktrees — **8**
`composer install` always runs in the laradock `workspace` container (PHP 7.4 parity), failing loudly if down; relativize `.git` links so a lane resolves on host **and** in-container (`/var/www`). *(PRD P0-9)*
- **AC** Container down → loud failure with start instructions, never host PHP.
- **AC** A lane's path maps 1:1 host↔container; `.git` link is relative.

### MT-11 · `[TALENTA]` Per-lane `*.local` hostname registration — **5**
On create/remove, register/unregister the deterministic hostname via `talenta-host`; a hostname failure must never abort the git op. `ls` shows the per-lane URL. *(PRD P0-10)*
- **AC** New lane is browsable at its `*.local` URL after create.
- **AC** `talenta-host` failure degrades to a warning; git op still succeeds.

---

## Phase 3 — Ergonomics & automation · ~21 SP

### MT-12 · `[AUTO]` `--json` machine output — **5**
Single result object on stdout `{type, platform, branch, base, lane, status}`; all human output → stderr; preserve `1>&2` on git-mutating calls. *(PRD P1-1)*
- **AC** `mwt --json bugfix TM-1` emits exactly one JSON line on stdout, nothing else.

### MT-13 · `[AUTO]` `claude` launcher — **5**
Resolve ticket/branch/suffix → lane (or cwd), `cd`, `exec claude` forwarding args; bypass the REPL alt-screen. *(PRD P1-2, P1-6)*
- **AC** `mwt claude TM-1 -p "run tests"` lands in the lane with args forwarded.
- **AC** `_resolve_lane` is most-specific-first; ambiguous → refuse + list.

### MT-14 · `[UX]` Interactive REPL — **8**
Bare `mtree`/`mwt` in a TTY → alt-screen banner + readline prompt; commands typed without `mwt`; each a guarded one-shot subprocess; `claude` bypasses. `mwt <cmd>` still one-shots for scripts. *(PRD P1-3)*
- **AC** REPL always restores the screen on exit/Ctrl-C (signal traps).
- **AC** Nested invocations guarded by `MWT_NESTED` (no REPL re-entry).

### MT-15 · `[AUTO]` `batch` / `[UX]` `scratch` / `resume` / `sync-env` — **3**
`batch <file>` (CRLF-tolerant, `#` comments); `scratch [name]` in `/tmp`; `resume <branch>`; `sync-env`. *(PRD P1-4/5/7, P0-1)*
- **AC** `batch` reports created/failed counts; bad rows don't abort the run.
- **AC** `resume` points to an existing checkout instead of erroring.

---

## Phase 3.5 — AI-assisted lane creation from a Jira URL · ~21 SP

> New feature *(not in v1.0.6)*. Depends on the Phase 1 engine **and** Phase 3's `claude`/`--json` plumbing. See [PRD.md](PRD.md) § *AI-Assisted Lane Creation from a Jira URL*.

### MT-19 · `[AUTO]` Jira-URL input parser + validation — **3**
Auto-detect a Jira URL in the ticket position of `<type> <jira-url>`; extract type, URL, key; validate type (registry), URL shape (`*.atlassian.net/browse/<KEY>`), and key (`^[A-Za-z]+-[0-9]+$`). *(PRD P1·AI-1)*
- **AC** `feature https://jurnal.atlassian.net/browse/WEB-1234` → type=`feature`, key=`WEB-1234`.
- **AC** Manual forms (`<type> <KEY> [desc]`) unchanged; only a URL triggers the AI flow.
- **AC** Unsupported type prints the `types` list; malformed URL prints the expected form.

### MT-20 · `[AUTO]` Claude-delegated retrieval + description generation — **8**
Shell out to `claude -p` (Atlassian MCP) to read the ticket in field-priority order (User Story → AC → Description → Summary → Labels) and return `{key, suggestion, rationale}`; mtree sanitizes to kebab-case regardless of model output. *(PRD P1·AI-2, P1·AI-3)*
- **AC** Suggestion reflects the implementation, not the summary verbatim; banned generics (`update/improve/fix-issue/changes/enhancement`) excluded.
- **AC** Output matches `^[a-z0-9]+(-[a-z0-9]+){1,5}$` after sanitization (lowercase, strip invalid chars, collapse/trim hyphens).
- **AC** `claude` missing/auth/MCP/timeout error → clear message + manual fallback `<type> <KEY> <desc>`.

### MT-21 · `[UX]` Interactive approval gate + manual edit — **5**
Preview (type, key, suggested description, full branch) with `[Y] accept · [E] edit · [C] cancel`; `[E]` replaces only the description (re-sanitized, re-previewed); type/key immutable. *(PRD P1·AI-4, P1·AI-5)*
- **AC** No git mutation before `[Y]`; `[C]` aborts with zero side effects.
- **AC** Non-TTY or `--yes`/`-y`/`--json` auto-accepts; non-TTY without a flag errors instead of hanging.

### MT-22 · `[CORE]` Engine wiring + result summary — **3**
On approval route `<prefix>/<KEY>-<desc>` + resolved base through `_provision_lane`; print branch, full path, base, status, and `cd`/`git status` follow-ups; `--json` extends `_emit` with suggestion fields. *(PRD P1·AI-6, P1·AI-7)*
- **AC** Partial failure aborts safely (never reported as success); existing branch/lane → idempotent point-there.
- **AC** `--web/--mobile`, `--base`, `--no-bootstrap` compose exactly as with manual `work`.

### MT-23 · `[AUTO]` Error taxonomy + troubleshooting log — **2**
Map every failure (invalid URL, unsupported type, ticket not found, Jira auth/unavailable, Claude unavailable, ambiguous ticket, worktree/branch exists, invalid name, git failure) to cause · resolution · retry; log parsed input, prompt, raw response, git steps (gated; `--json` stdout stays pure). *(PRD P1·AI-8, P1·AI-9)*
- **AC** Ambiguous/low-signal ticket still offers `[E]` rather than failing; transient errors advise retry, deterministic ones don't.

### MT-24 · `[AUTO]` *(P2·AI, future)* Suggestion cache · base-branch inference · alternatives — **3**
Cache suggestions per ticket; infer base from labels (offer, don't auto-apply); offer 2–3 candidate descriptions. *(PRD P2·AI)*

---

## Phase 4 — Reach (P2, not scheduled) · ~8 SP

### MT-16 · `[CORE]` Non-APFS reflink fallback (Linux) — **5** *(P2-1)*
### MT-17 · `[CORE]` Additional stack detectors (Node-only, Python) — **2** *(P2-2)*
### MT-18 · `[TALENTA]` Pluggable host/Docker providers via `wt.*` — **1** *(P2-3)*

---

## Cross-cutting (every phase)

### MT-QA · `[QA]` Smoke-test suite (`bats`) for risky paths — **5**
Today the only gate is `bash -n`. Add tests for `cleanup` merge-detection, `remove` dirty-guard, COW-clone fallback, and `--json` purity. *(PRD Open Question [eng])*
- **AC** CI runs `bash -n` + `bats` on every change; the four risky paths are covered.

### MT-DOCS · `[DOCS]` README + published guide + `help` parity — **3**
Keep `mtree help`, README, and the GitHub Pages guide in sync; document `wt.*` config and the lane model.
- **AC** Every shipped command appears in `help` and the guide.

### MT-TEL · `[AUTO]` Opt-in telemetry — **3** *(PRD Open Question [data])*
Emit anonymized counters (lane creates, bootstrap success, context-switch timing) so the Success Metrics are measurable rather than asserted.
- **AC** Telemetry is opt-in, documented, and produces the metrics in PRD § Success Metrics.

---

## Dependency order

```
MT-1 → MT-2 → MT-3 → MT-4 → {MT-5, MT-6, MT-7}        (Phase 1)
MT-3 ─────────────→ MT-8 → MT-9 → MT-10 → MT-11        (Phase 2, needs the engine)
MT-3 ─────────────→ MT-12 → MT-13 → MT-14 → MT-15      (Phase 3)
{MT-3, MT-13, MT-12} → MT-19 → MT-20 → MT-21 → MT-22 → MT-23   (Phase 3.5, needs engine + claude + --json)
MT-QA / MT-DOCS / MT-TEL run alongside from Phase 1
```
