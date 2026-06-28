# mtree — One-Page Pitch

> **Mekari Git Worktree** · turning a 3-minute context switch into one command · *for: eng leads, platform/DX owners, anyone funding developer-experience time*

---

## The problem, in one sentence

A monorepo checkout is a single desk — every PR review, hotfix, or parallel ticket in `talenta-core` costs a `git stash` + `checkout` + minutes-long Composer/Yarn rebuild, **twice** (there and back) — and AI agents now want their own clean desk too.

## What it costs today

- **~3–5 min per context switch**, several times a day, per engineer. At ~30 engineers that's hours of dead time daily.
- **Broken local environments** when a half-switched checkout or a hand-rolled `git worktree` leaves deps/env corrupt.
- **A ceiling on parallelism** — you can't easily run two tickets, or dispatch two Claude Code agents, against one desk without them racing each other.

## The solution

`mtree` wraps `git worktree` into one-command "lanes" — a handful of long-lived working directories, each pinned to a role (`main`, `mobile`, per-ticket `work`/`hotfix`, `review`, `scratch`), all sharing one `.git`.

```
mwt bugfix TM-1 npe     # isolated branch + deps ready, in seconds
mwt review feature/X    # read a teammate's PR without touching your work
mwt claude TM-1         # launch an AI agent in its own clean lane
mwt cleanup             # merged lanes self-garbage-collect
```

## Why it's fast and safe (the one non-obvious idea)

Dependencies are **cloned, never symlinked**. When a lane's lockfile matches main, `vendor/` and `node_modules/` are materialized with an **APFS copy-on-write reflink** — near-instant, ~0 disk until they diverge, fully isolated. Symlinking was deliberately rejected: it would let one lane's build cache corrupt the pristine main checkout and let two agents race one directory. *This is the difference between "parallel work that's safe" and "parallel work that occasionally eats your environment."*

## What we'd be funding (if rebuilt from scratch)

| Phase | Scope | Outcome |
|---|---|---|
| **1 — Core** | Lane engine, data-driven ticket types, clone-or-install bootstrap, `ls`/`remove`/`cleanup`/`doctor` | Works in *any* repo |
| **2 — talenta-core** | Web/Mobile split, composer-in-laradock (PHP parity), per-lane `*.local` URLs | Tuned for our monorepo |
| **3 — Ergonomics** | `--json`, `claude` launcher, interactive REPL, `batch` | Agent- & automation-ready |
| **4 — Reach** | Linux reflink fallback, more stacks, telemetry | Beyond Mac/PHP |

## The ask

Treat mtree as funded DX infrastructure: a maintainer owner, a lightweight smoke-test suite (none today), and **opt-in telemetry** so we can prove the wins below instead of asserting them.

## How we'll know it worked

- **Context switch: minutes → < 30 s** (warm) / one command.
- **Adoption: ≥ 80%** of active talenta-core engineers within 30 days.
- **Corrupted-checkout incidents → ~0**; **disk footprint stays flat** despite N lanes.
- **Agent throughput up** — more parallel Claude Code tickets per dev, no local-env breakage.

## Cost, risk, and why now

- **Cost is low:** it's one self-contained Bash file — no build, no deps, no server. The real cost is a maintainer's attention.
- **Risk is contained:** it's a thin wrapper over `git worktree`; users can always drop to raw git. It never pushes, deletes remotes, or opens PRs.
- **Why now:** AI agents make per-task isolated checkouts a *throughput multiplier*, not just a convenience — and the tool already exists and is in daily use. We're formalizing and resourcing what's already proven, not betting on a prototype.
