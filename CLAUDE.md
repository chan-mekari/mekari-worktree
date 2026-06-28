# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`mtree` (a.k.a. "Mekari Git Worktree") is a **single self-contained Bash script** (`mtree`, ~1000 lines) that wraps `git worktree` into one-command operations. It implements matklad's "concurrency lane" model — a few long-lived working directories, each with a fixed role, all sharing one `.git` object store — tailored to the **`talenta-core`** monorepo (PHP/Laravel + Vue, run under a `laradock` Docker stack).

Two framing facts that change how you work here:

- **This directory is not a git repo** — it is the home of the tool. `mtree` discovers and operates on *whatever git repo it is invoked inside* (`git rev-parse` gate at mtree:106). Most commands work on any repo; only `cmd_init`/`cmd_mobile` and the `--web/--mobile` platform split are `talenta-core`-specific.
- **The entire program is one file.** There is no build system, package manager, test suite, dependency manifest, or CI config. Don't look for `package.json`, `Makefile`, or a test runner — they don't exist.

## Commands

```bash
./mtree <command> [args]      # run directly
bash -n mtree                 # syntax check — the only "build"/lint gate that exists
shellcheck mtree              # optional static analysis if available (not required/configured)
```

There is **no automated test suite**. Verify changes by syntax-checking (`bash -n mtree`) and exercising the affected command against a throwaway repo. `cmd_init` installs two shell aliases:

- `mtree` → opens the interactive REPL (alt-screen banner + prompt).
- `mwt <cmd>` → runs one command and exits (what scripts/aliases use).

Common operations (see `./mtree help` for the full list): `work <type> <TICKET> [desc]`, `<type> <TICKET>` (every type is also a direct subcommand), `resume <branch>`, `hotfix [web|mobile] <TICKET>`, `review <branch>`, `remove <TICKET>`, `cleanup`, `ls`, `doctor`, `types`, `claude [target]`.

## Architecture

It is one file, so "architecture" here means the cross-cutting patterns you must understand before editing any one part.

### Dispatch + the type registry (the core design)

The bottom of the file (mtree:965–1018) parses global flags out of `$@`, then `case`-dispatches on the first positional. The key idea: **ticket types are data, not code.** The `TCW_TYPES` heredoc table (mtree:121) lists rows `type|prefix|web-base|mobile-base|mode`. A single engine — `cmd_work` → `_provision_lane` — routes every type through it. The dispatch `*)` fallthrough (mtree:1016) treats any unrecognized command as a possible type via `_type_row`, so **adding one row to `TCW_TYPES` instantly makes that type work both as `work <type> <TICKET>` and as a bare `<type> <TICKET>` subcommand.** This is the first thing to understand: feature changes are usually table edits, not new code paths.

Note the table heredoc is intentionally **unquoted** so `${TCW_BASE}`/`${TCW_WEB_PROD}`/`${TCW_MOBILE_PROD}` expand into every row — each type's base branches follow the repo's resolved bases.

### The lane model

All tool-created worktrees live under one container next to the main checkout: `<parent>/<repo>-worktrees/<repo>-<suffix>` (`WT_ROOT` / `wt_dir`, mtree:137). Lanes: `main` (the clean checkout, never removed), `mobile` (persistent, off the mobile base), `work`/`hotfix` (one per ticket, parallel), `review` (detached, one branch at a time), `scratch` (`/tmp`, throwaway). `_resolve_lane` (mtree:217) maps a user token → directory most-specific-first: `main` → exact branch checked out → fuzzy ticket match → conventional slugified lane dir.

### Config resolution (per-repo, layered)

Every tunable resolves `git config wt.<key>` → environment variable → inferred default (e.g. `_resolve_base` at mtree:73, and the `wt.web`/`wt.mobile`/`wt.docker*` reads at mtree:119–147). Pin per repo with e.g. `git config wt.base develop`. Repo discovery itself uses `git worktree list --porcelain` — the first entry is `MAIN` — so the tool works from inside *any* worktree.

### Dependency bootstrap — clone-or-install, never symlink

A fresh worktree has no `.env`, `vendor/`, or `node_modules`. `bootstrap` (mtree:323) selects per stack by which manifest `MAIN` has:

- **Go service** (`go.mod` present): `go mod download` + `go mod vendor` on the host; **no** composer/yarn even if a `yarn.lock` exists.
- **PHP (`talenta-core`)**: `composer` + `yarn` via `_provision` (mtree:271). For each dep dir: if the lockfile hash matches `MAIN`, do an **APFS copy-on-write clone** (`cp -c -R`) — isolated and near-instant; otherwise a real install. **Symlinking these dirs is deliberately rejected** (post-install hooks `cp -rf` into them, yarn rewrites `.yarn-integrity`, builds write `node_modules/.cache` — through a symlink all of that would corrupt the pristine main checkout and two agents would race on one directory). Only `.env` is copied (it's gitignored); `.env.bitbucket`/`.env.example` are tracked, so git materializes them automatically.

### Docker / laradock coupling

`composer install` **always runs inside the laradock `workspace` container** (PHP 7.4 parity), never on host PHP — `_composer_install_in_container` (mtree:259) fails loudly if the container isn't up. laradock mounts `$PARENT` at `/var/www`, and `_relativize_worktrees` (mtree:176) rewrites each worktree's `.git` link to a relative path so a lane resolves both on the host (`/Users/...`) and in the container (`/var/www/...`). `_docker_path` maps host lane paths → container paths 1:1.

### talenta-host integration (talenta-core only)

On lane create/remove, `mtree` registers/unregisters a deterministic `*.local` hostname (nginx vhost + `/etc/hosts` + `APP_URL`) by shelling out to the companion `talenta-docker-dev/bin/talenta-host` script (`_register_host`/`_unregister_host`, mtree:163). A hostname hiccup must **never** abort the underlying git operation — those calls always swallow failure with a warning.

### Output modes — human vs `--json`

When `--json` is set, all human-readable output goes to **stderr** so stdout stays pure JSON. This is why `_say`/`log`/`info`/`ok` branch on `OPT_JSON`, why `die`/`warn` always write to stderr, and why git-mutating calls throughout carry `1>&2`. `_emit` (mtree:245) prints the single JSON result line. **Preserve these redirects when editing** — dropping a `1>&2` corrupts `--json` consumers (e.g. the daily-driver automation).

### Interactive REPL vs one-shot

A bare `mtree`/`mwt` in a TTY enters a full-screen alt-screen session (`_enter_interactive` at mtree:834): banner + readline prompt where commands are typed *without* the `mwt` prefix, each run as a fresh subprocess guarded by `MWT_NESTED=1` to prevent REPL re-entry. `mwt <cmd>` runs once and exits. **`claude` is the exception** — it hands the TTY to a full-screen app, so it bypasses the alt-screen REPL (nesting two alt-screens corrupts the display) and falls straight through to dispatch (mtree:989).

## Conventions when editing

- `set -euo pipefail` is in force. **`die()` inside `$(...)` only exits the command-substitution subshell**, so callers that capture output guard with `|| exit 1` (see `_resolve_lane` usage). Keep this pattern.
- Naming: `cmd_*` = user-facing command handlers; `_lowercase` = internal helpers. Avoid `local type=` — it shadows the `type` builtin; this codebase uses `wtype` deliberately.
- macOS / BSD assumptions are load-bearing: APFS `clonefile` (`cp -c`), `stty size </dev/tty` for real terminal width (`tput cols` breaks inside `$()`), and BSD `awk` forbids newlines in `-v` values (the `ls` table comma-joins URLs to work around it).
- `self()` returns `mwt` — the display name used in all help/error text. `TCW_VERSION` (mtree:70) is the banner version string.
- `docs/reviews/` holds generated code-review reports (output of the review tooling), not source.
