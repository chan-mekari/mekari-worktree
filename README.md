<p align="center">
  <img src="assets/mtree-256.png" alt="mtree icon" width="120" height="120">
</p>

<h1 align="center">mtree — Mekari Git Worktree</h1>

`mtree` wraps `git worktree` into one-command operations. It implements
[matklad's "concurrency lane" model](https://matklad.github.io/2024/07/25/git-worktrees.html) —
a few long-lived working directories, each with a fixed role, all sharing one `.git` object
store — tuned for the **`talenta-core`** monorepo (PHP/Laravel + Vue, run under a `laradock`
Docker stack).

The entire program is a **single self-contained Bash script** (`mtree`). There is no build
step, package manager, or dependency manifest. Most commands work in any git repo; a few
(`init`, `mobile`, and the `--web`/`--mobile` platform split) are `talenta-core`-specific.

## Documentation

A full guide — overview, the lane model, setup, daily workflow, the command reference,
security practices, an FAQ, and how to collaborate — is published via GitHub Pages:

**→ <https://chan-mekari.github.io/mekari-worktree/guide.html>**

The source lives at [`docs/guide.html`](docs/guide.html).

## Requirements

- **Bash** and **git** (uses `git worktree`).
- **macOS / APFS** assumptions are load-bearing — APFS copy-on-write clones (`cp -c`),
  BSD `awk`, and real-terminal sizing via `stty`.
- For `talenta-core` dependency bootstrap: a running **laradock** stack (composer installs
  run inside the `workspace` container for PHP parity), plus `yarn` for the frontend.

## Install

```bash
# Place the script on your PATH (or symlink it), e.g.:
ln -s "$PWD/mtree" /usr/local/bin/mtree

# One-time, idempotent setup inside your repo (talenta-core): installs aliases + the Mobile-API lane
mtree init
```

`init` wires up two shell aliases:

- **`mtree`** → opens the interactive REPL (full-screen banner + prompt; type commands without a prefix).
- **`mwt <cmd>`** → runs one command and exits — what scripts and day-to-day aliases use.

Run `mtree help` for the full command list.

## Common commands

```bash
mwt work <type> <TICKET> [desc]   # create <prefix>/<TICKET>[-desc] off its base, then bootstrap deps
mwt <type> <TICKET> [desc]        # every type is also a direct subcommand, e.g. `mwt bugfix TM-1 npe`
mwt resume <branch> [suffix]      # continue an existing branch (local or origin) in its own lane
mwt hotfix [web|mobile] <TICKET>  # hotfix/<TICKET> off master / master-mobile-api (defaults to web)
mwt review <branch>               # detached lane at origin/<branch> (pass the PR source branch)
mwt scratch [name]                # throwaway detached worktree in /tmp
mwt claude [target] [args…]       # launch Claude Code in a lane (omit target → current dir)

mwt ls                            # list all worktrees
mwt remove <TICKET> [--force]     # remove a ticket's lane (keeps the local branch; never touches remotes)
mwt cleanup                       # remove clean lanes already merged into the base branch
mwt doctor                        # health report: disk, dep drift, prunable/locked lanes
mwt types                         # list the ticket-type registry
```

Useful flags: `--base <branch>` (override the base branch), `--web` / `--mobile`
(platform base — `talenta-core` only), `--no-bootstrap` (skip dep provisioning),
`--json` (emit a result object on stdout), `--force`.

### Ticket types

Types are **data, not code** — a registry table near the top of the script. Adding one row
makes that type work both as `work <type> <TICKET>` and as a bare `<type> <TICKET>`
subcommand. Built-in types: `bugfix`, `subtask`, `fasttrack`, `task`, `story`,
`improvement`, `techdebt`, `feature`, `epic`, `spike`. Run `mwt types` to see each type's
prefix and base branches.

## The lane model

All tool-created worktrees live under one container directory next to the main checkout:

```
<parent>/<repo>-worktrees/<repo>-<suffix>
e.g.  ~/www/talenta-core-worktrees/talenta-core-bugfix-TM-1
```

| Lane      | Role                                                      |
|-----------|-----------------------------------------------------------|
| `main`    | the clean primary checkout — never removed                |
| `mobile`  | persistent lane off the mobile base (`talenta-core`)      |
| `work`    | one per ticket, created on demand, run in parallel        |
| `hotfix`  | one per ticket, branched off the production base          |
| `review`  | detached, one branch at a time (for reviewing a PR)       |
| `scratch` | throwaway lane in `/tmp`                                  |

Dependencies are **cloned or installed, never symlinked**: when a lane's lockfile matches the
main checkout, deps are materialized with an APFS copy-on-write clone (isolated and
near-instant); otherwise a real install runs. Only `.env` is copied.

## Configuration

Every tunable resolves in this order: **`git config wt.<key>` → environment variable →
inferred default**. Pin a setting per repo, for example:

```bash
git config wt.base develop          # the base branch new lanes branch from
git config wt.dockerDir <path>      # laradock directory (talenta-core)
git config wt.dockerService workspace
```

## Development

It's one file. Verify changes with the only "build"/lint gate that exists:

```bash
bash -n mtree        # syntax check
shellcheck mtree     # optional static analysis, if installed
```

`CLAUDE.md` documents the internal architecture in depth (dispatch + type registry, the lane
model, config resolution, dependency bootstrap, Docker/laradock coupling, and the human-vs-`--json`
output split).
