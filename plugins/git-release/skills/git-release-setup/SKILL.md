---
name: git-release-setup
description: >-
  Use when installing or configuring `git-release`, initializing release
  versioning in a repo, or when a git-release command fails before it does any
  work — "command not found", "No Release is Initialized", a release file that
  cannot be written, the wrong branch treated as main, an
  `afterversioncommit.sh` hook aborting a cut, or a candidate number out of
  sync with the branches that exist. Covers install and upgrade, `init`, the
  `releases.*` config keys and their defaults, the version hook contract,
  feature-branch helpers, cleanup, and a troubleshooting map.
user-invocable: true
allowed-tools: Bash, Read, Grep, Glob
argument-hint: "[what you are setting up or the error you hit]"
---

# git-release-setup — install, configure, troubleshoot

## 1. Install

`git-release` is one Bash script. It needs `git`, `bash`, and `curl` — nothing
else, no build step.

```bash
curl -s -L https://raw.githubusercontent.com/neverbehind/git-release/main/install.sh | bash
```

`install.sh` drops the script at `~/bin/git-release`, `chmod +x`es it, and
appends `~/bin` to `PATH` in `~/.zshrc` or `~/.bashrc`. From a repo checkout,
`bash install.sh` does the same. Because it lands on `PATH` as `git-release`,
git's subcommand dispatch makes `git release <cmd>` work; `./git-release <cmd>`
runs a checkout copy directly.

```bash
git release upgrade      # re-downloads main over ~/bin/git-release
git release help         # command inventory (stale in two places — see §6)
```

There is no version command. To identify a build, check the checkout:
`git -C <checkout> log -1 --oneline git-release`. To test for the one capability
that changes how you may drive the tool — whether prompts are guarded against
running without a terminal:

```bash
git release help | grep -q '\[tty\]' && echo guarded || echo unguarded
```

An unguarded build will act on an empty answer rather than refusing. See §8 of
the `git-release` skill before scripting anything against one.

## 2. Initialize a repo

```bash
git release init 1.4.0 0     # both args -> non-interactive
```

Missing arguments make `init` prompt. **Always pass both from an agent
session** — on a build without the interactive guard, an `init` that reads EOF
instead of an answer writes `releases.version=""` and
`releases.current=release-v` and exits 0, and the next `roll` builds on that.
Guarded builds exit 78 and write nothing.

It writes `releases.version`, `releases.candidate`, and
`releases.current` (= `release-v<version>`), **clears the branch list**, then
prints `status`. Candidate `0` means the first `roll` produces `rc1`.

`init` is also called internally by `checkout` and `duplicate`, which is why
switching releases wipes and re-reads the branch list.

## 3. The config keys

All under `releases.*` in `.git/config`, all per-clone:

| Key | Default | Set with |
|---|---|---|
| `mainbranch` | `main` (written on first read) | `git release setmainbranch <name>` |
| `stagebranch` | `staging` (written on first read) | `git release setstagebranch <name>` |
| `qabranch` | `qa` (written on first read) | `git release setqabranch <name>` |
| `devbranch` | **none** — prompts when needed | `git release setdevelopbranch <name>` |
| `devdeployurl` / `stagedeployurl` / `qadeployurl` | unset | `git release set*deployurl <url>` |
| `version` / `candidate` / `current` / `branches` | set by `init` / `add` | see the `git-release` skill |

The three branch defaults are **written into config the first time they are
read**, so a repo whose trunk is `master` silently gets `releases.mainbranch=main`
unless you set it before the first command. Set the branch names immediately
after `init`:

```bash
git release init 1.4.0 0
git release setmainbranch master
git release setstagebranch staging
git release setqabranch qa
git release setdevelopbranch develop
git config --local --get-regexp '^releases\.'     # confirm
```

Called without a value, the `set*deployurl` commands prompt; `setmainbranch` and
friends called without a value silently store an empty string. Always pass the
value.

`releases.stagebranch` and `releases.qabranch` always resolve, defaulting to
`staging`/`qa` on first read, so the "no staging branch configured" prompts in
`stagebranches`/`qabranches` are unreachable. `releases.devbranch` has no
default, so `devbranches` really does prompt — set it up front.

Config is local to the clone. A fresh clone, or a colleague's machine, has no
release state — `git release init`/`checkout` reestablishes it. `releases/<name>`
committed on the RC branch is the portable backup of the branch list; `readin`
restores it (and hard-resets the tree, so human-only).

## 4. The `afterversioncommit.sh` hook

Put an executable `afterversioncommit.sh` in the repo root. `roll`, `next`, and
`append` run it with `bash` immediately after committing the release file and
`version` marker, before merging any feature branch. Its job is propagating the
version into other manifests — `package.json`, `composer.json`, a `.env`.

Contract:

- The new RC branch name is already written to the `version` file in the repo
  root. Read it from there; nothing is passed as an argument.
- The hook **must commit its own changes** — the tool does not commit after it.
- **Exit 0** to continue. **Any non-zero exit aborts the whole cut with
  `exit 1`** — except exit code **1 during `append` only**, which is tolerated
  and logged as the "nothing to commit" case.
- It runs before the feature merges, so a hook that fails leaves you on a new RC
  branch containing only the version commit, with the candidate number already
  bumped. See §6.

Minimal shape:

```bash
#!/bin/bash
VERSION=$(sed 's/^release-v//; s/-rc[0-9]*$//' version)
npm version "$VERSION" --no-git-tag-version --allow-same-version
git add package.json package-lock.json
git commit -m "Bump version to $VERSION" || exit 1   # exit 1 is tolerated on append
```

## 5. Feature-branch helpers

Convenience wrappers around plain git. **All of them prompt**, so they are for a
human at a keyboard, not an agent. On a guarded build they exit 78 without
acting; on an older build they hang, or (at EOF) run partway on empty answers:

- `newfeature` — menu-driven prefix (`feature/`, `bugfix/`, `hotfix/`), checks
  case-insensitively for an existing branch, then branches from
  `origin/<mainbranch>` and pushes with `-u`.
- `checkoutfeature` — search remote branches, create the local tracking branch.
- `pushfeature` — the long chain: push the current branch, merge it to develop,
  offer to roll/next/append the RC, offer to stage. Refuses to run on
  main/stage/qa/dev.
- `devfeature <branch>` — merge one feature into the develop branch and push.
- `updatelocal` — fast-forward every local branch that is behind, stashing first
  if asked.

The agent equivalents are ordinary git: `git checkout -b feature/x
origin/main && git push -u origin feature/x`, then `git release add
origin/feature/x`.

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `git: 'release' is not a git command` | `~/bin` not on `PATH`, or install never ran | `ls -l ~/bin/git-release`; re-run `install.sh`; open a new shell |
| `No Release is Initialized` from `add` | `releases.current` unset in this clone | `git release init <version> <candidate>` |
| `No Release Found` banner from `status` | same, or you are in the wrong repo | check `git rev-parse --show-toplevel` |
| Release file not written by `next`/`append` | `releases/` does not exist; only `roll` creates it | run `roll` once, or `mkdir releases` and commit |
| Candidate number ahead of any real branch | `roll`/`next` bump the candidate *before* doing the work; the cut then failed | `git release setcandidate <n>` to point at the RC that exists |
| Cut aborts right after the version commit | `afterversioncommit.sh` exited non-zero | run `bash afterversioncommit.sh` by hand and read its output; then `setcandidate` back and re-cut |
| Wrong trunk used everywhere | `releases.mainbranch` defaulted to `main` on first read | `git release setmainbranch <name>`, then `roll` |
| A command prompts and an unattended run hangs | stdin is open but nobody is typing | see §8 of the `git-release` skill; use the non-interactive equivalent |
| A command "succeeded" but state is wrong | on an unguarded build `read` hit EOF and the empty answer was used as the value | `git config --local --get-regexp '^releases\.'` to see what was written; `git release upgrade` to get the guard |
| `exit 78` from a command | guarded build: it needs an answer typed at a terminal | run it interactively, or use the non-interactive equivalent; `GIT_RELEASE_ASSUME_TTY=1` only for deliberate piping |
| `exit 64`, "not a git release command" | guarded build: the name is not a defined command | check it against `git release help` |
| A typo ran a shell command | older builds dispatch a bare `$COMMAND "$@"` | `git release upgrade`; never pass an untrusted string as the subcommand |
| `deploy` reset the branch you were on | older build, `deploy` run with no environment argument | recover with `git reflog`; always pass the env, and upgrade |
| `git release reset` ate uncommitted work | it is `git reset --hard`; the help text is wrong | recover from `git reflog` / `git stash list` if anything was staged |
| Merged branch shows as unmerged in `status` | the check uses **local** main | `git fetch --all && git checkout <main> && git pull`, or use `git merge-base --is-ancestor` |

Two documentation bugs in older `git release help` text, both since corrected:
`reset` was described as clearing release config (it hard-resets the working
tree instead), and `rm` was shown as taking a branch argument (it ignores
arguments and opens a picker — `remove <branch>` is the one that takes a ref).
If your help output still says either, you are on an old build: the behavior is
as described here, not as the help claims.

## 7. Cleanup

```bash
git release cleanup                       # prompts through the three below
git release cleanrelease [release-vX.Y.Z] # delete old RCs of a release, local + origin
git release cleanupmergedlocalbranches    # delete local branches merged into main
git release cleanupmergedremotebranches   # delete remote branches merged into main
git release purgelocalbranches            # delete every local branch except main
git release dump                          # delete the current RC and roll the candidate back
```

All of these delete branches, most of them on `origin`. Every one except
`purgelocalbranches` prompts per branch — and `purgelocalbranches` is the one
that does not ask at all (it uses `git branch -d`, so unmerged branches are
refused; everything else goes). `dump` requires a literal uppercase `Y`.

Two traps in the two `cleanupmerged*` commands:

- **`auto` is not "yes to this one".** `CHOICE` persists across loop
  iterations, so answering `auto` once deletes **every remaining branch** with
  no further prompt. Never pipe it in.
- **On older builds the "in the current release" protection never fires.**
  `cleanupmergedremotebranches` compares `remotes/origin/<name>` against
  `releases.branches`, which stores `origin/<name>`, so branches in the active
  release are offered for deletion like any other. Verified. Diff the deletion
  list against `git release branches` yourself.

Confirm with the user before running any of them, and never run
`cleanupmergedremotebranches` without first checking that the release has
actually merged back to main (`git merge-base --is-ancestor` against
`origin/main`, not the tool's local-main comparison).

## 8. Verify before "done"

- After install: `command -v git-release && git release help | head -3`.
- After `init` / `set*`: `git config --local --get-regexp '^releases\.'`.
- After a hook change: run `bash afterversioncommit.sh` in a scratch clone and
  check `echo $?` before trusting it in a cut.
- Anything you did not read back is a **carried** claim; say so.
