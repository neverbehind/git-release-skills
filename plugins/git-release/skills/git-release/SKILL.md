---
name: git-release
description: >-
  Use whenever a task touches release candidates managed by the `git-release`
  CLI — cutting an RC, adding or removing feature branches from a release,
  choosing between `roll` / `next` / `append`, reading release status
  ("what's in this release?", "has it shipped?", "which RC are we on?"),
  resolving a halted merge, or cleaning up old release branches. Covers the
  git-config state model that makes `.git/config` the source of truth, the
  three ways to cut a candidate and when each is wrong, and the guardrails
  that prevent the silent failures (interactive prompts that hang an agent or,
  worse, act on an empty answer; a bumped candidate after a failed roll;
  `reset` discarding your work).
user-invocable: true
allowed-tools: Bash, Read, Grep, Glob
argument-hint: "[what you are trying to do with the release]"
---

# git-release — release candidates from Claude Code

`git-release` is a single Bash script on `PATH` (installed to `~/bin/git-release`),
so `git release <command>` and `git-release <command>` are the same thing. It
manages **release candidate branches**: `release-v<version>-rc<n>`, built by
merging a configured list of feature branches on top of the main branch.

The whole model is three facts:

1. **State lives in `.git/config`**, under `releases.*`. Not in a file the tool
   parses, not on a server. Per-clone and per-repo.
2. **An RC branch is disposable output.** It is rebuilt from the branch list;
   nothing should be authored directly on it except conflict resolutions.
3. **The branch list is the input.** Change the list, re-cut the RC.

`git release help` is authoritative for the command inventory — but its text is
stale in two places (see §7). This skill tells you *which* commands matter,
*what they mutate*, and *which ones will hang an unattended session*.

Deployment (`to` / `stage` / `qa` / `deploy` / `merge` / `tag`) lives in the
`git-release-deploy` skill. Install, branch configuration, the
`afterversioncommit.sh` hook, and troubleshooting live in `git-release-setup`.

## 1. Orient before you touch anything

```bash
git release status          # version, RC, branch list, configured env branches
git config --local --get-regexp '^releases\.'   # the raw state, no side effects
git symbolic-ref --short HEAD                   # where you're standing
git status --porcelain                          # uncommitted work at risk
```

`status` is the one read command that is safe everywhere: it prints and returns
without checking out anything. **`deploystatus` is not** — it calls
`releasebranches`, which checks out and pulls the RC branch. Never run
`deploystatus` over a dirty tree.

Almost every other command **changes the checked-out branch and leaves you
there**. `roll` leaves you on the new RC; `stage`/`qa`/`to` leave you on the
deploy target; `cleanrelease` and the cleanup commands leave you on main. Always
record the starting branch and restore it when you're done.

Exit codes are usable but narrow. `roll`/`next`/`append` return 1 when a merge
conflicts and correctly skip the push (verified), but they return **0 even if
the push itself fails** — a zero exit is not evidence the RC reached origin.
See §9.

## 2. The state model

```
releases.version     1.4.0                 -> tag will be v1.4.0
releases.candidate   3                     -> current RC number
releases.current     release-v1.4.0        -> release name
releases.branches    origin/feature/login  -> multi-value; the input list
releases.branches    origin/bugfix/M2S-77
releases.mainbranch  main                  (defaults: main / staging / qa)
releases.stagebranch staging
releases.qabranch    qa
releases.devbranch   develop               (no default — must be set)
```

Derived, not stored: the release branch is `$(current)-rc$(candidate)`, i.e.
`release-v1.4.0-rc3`. Read the pieces with `git release version`, `candidate`,
`current`, `branches` — each prints one value and mutates nothing.

Two files are committed onto the RC branch by every cut, as a backup of the
list and a build marker:

- `releases/<current>` — the branch list, one per line.
- `version` — the RC branch name.

`releases/` is created only by `roll` (via `migratereleases`). **`next` and
`append` do not create it**, so on a repo that has never rolled they fail to
write the release file. Roll once first.

## 3. Build the branch list

Use the **non-interactive** pair. `add` takes a full remote ref exactly as it
should be merged:

```bash
git release add origin/feature/login       # appends to releases.branches
git release remove origin/feature/login    # exact string match, then reprints status
git release branches                       # the current list
```

- `add` does **not** deduplicate. Adding the same ref twice merges it twice —
  harmless but noisy in the log. Check `branches` first.
- `remove` matches the stored string exactly. `feature/login` will not remove
  `origin/feature/login`.
- `add` refuses (`exit 1`) when no release is initialized. Run
  `git release init <version> <candidate>` first — see `git-release-setup`.
- `add` with **no** argument stores an empty entry on older builds (exit 0, no
  complaint); guarded builds reject it. Either way, always pass the ref.
- Whatever you store is passed straight to `git merge`, so it must be a ref that
  resolves after `git fetch --all`. Prefer `origin/<branch>` over a local name:
  a local branch can be stale, and the merge will silently use the stale tip.

`git release feature <search>` and `git release rm` do the same work through a
numbered picker, and both need a terminal (§8). Use them only when a human is at
the keyboard; from an agent session, resolve the branch yourself and call
`add`/`remove`:

```bash
git fetch --all
git branch -r | grep -i login              # pick the ref yourself
git release add origin/feature/login
```

## 4. Cut a candidate: roll vs next vs append

All three write the release file + `version`, commit, run
`afterversioncommit.sh` if present, merge `origin/<mainbranch>`, then merge every
branch in the list in order, then push **only if there are no conflicts**.

| Command | Base | Candidate number | Use when |
|---|---|---|---|
| `roll` | fresh `main` (checkout + pull) | **bumped** | first RC of a release; **any time a branch was removed from the list**; the list changed materially |
| `next` | the current RC | **bumped** | carrying an RC forward with a fix; you want a clean diff between rc(n) and rc(n+1) |
| `append` | the current RC, in place | unchanged | small re-merge onto the RC that is already deployed somewhere |

The decision that actually matters: **removing a branch from the list requires
`roll`.** `next` and `append` build on the existing RC, which already contains
the removed branch's merge commit — dropping it from the list does not unmerge
it. Only re-cutting from main does.

Mechanics worth knowing:

- **`roll` and `next` bump `releases.candidate` before doing any work**, as a
  side effect of computing the next branch name. If the cut then fails (conflict,
  hook failure, bad ref), config already points at an RC that may not exist.
  Recover with `git release setcandidate <n>`, or `git release dump` to delete
  the branch and roll the number back.
- `append` prints an error from `git checkout -b <existing branch>` and then
  checks the branch out normally. That error is expected; ignore it.
- `afterversioncommit.sh` returning 1 aborts `roll` and `next` but is tolerated
  during `append` (it is the "nothing to commit" case). Any other non-zero exit
  aborts all three with `exit 1`.
- `git release push` pushes the current RC — the manual follow-up after you
  resolve conflicts.
- **Nothing checks for a dirty tree.** `roll` runs `git checkout <main>` and
  carries uncommitted changes onto the new RC, where the release-file commit can
  sweep them in; if the checkout is refused outright, the script keeps going and
  cuts the RC from wherever you were standing. Check `git status --porcelain`
  before any cut.

## 5. When a cut halts on conflicts

The tool merges everything it can, then reports. It does **not** abort the merge
and does **not** push. You are left on the RC branch with unmerged paths.

```bash
git diff --name-only --diff-filter=U      # exactly what the tool checked
# resolve the files
git commit -a -m "resolved conflicts"
git release push
```

Do not re-run `roll` to "retry" — that bumps the candidate again and re-hits the
same conflict from a clean base. If two branches conflict every cycle, the fix
is upstream: merge the two interdependent branches together and add *that* to the
release instead.

A conflict resolved on rc(n) survives into rc(n+1) via `next` or `append`, but
**not** through `roll`, which rebuilds from main.

## 6. Reading what shipped

```bash
git release status              # includes a Pending/Deployed line
git release deploystatus        # status + each release branch not yet in main, with SHA
git release releasebranches     # branches in the RC that are not yet in main
git release devbranches         # in develop, not in main   (prompts if devbranch unset)
git release stagebranches       # in staging, not in main
git release qabranches          # in qa, not in main
```

The `Status: Deployed` line and `releasebranches` compare against **local**
main, not `origin/main`. A stale local main reports a release as pending after it
has actually shipped. Refresh first (`git fetch --all && git checkout main &&
git pull`) or check directly:

```bash
git fetch --all
git merge-base --is-ancestor "$(git release releasebranch)" origin/main && echo shipped
```

`devbranches`/`stagebranches`/`qabranches` all check out and pull that branch as
a side effect, and `devbranches` prompts for a branch name if `releases.devbranch`
is unset. Set it up front (`git release setdevelopbranch develop`) rather than
discovering the prompt mid-run.

## 7. Guardrails

- **`git release reset` is `git reset --hard`.** It discards your uncommitted
  work and leaves config untouched. To actually clear release state, unset the
  keys yourself: `git config --local --remove-section releases`. Older builds'
  help describes it as "Clear all release config from git config" — that text
  is wrong; believe the behavior.
- **`git release rm` takes no argument.** Older help shows `rm <branch>`; the
  command ignores it and opens a picker. `remove <branch>` takes the ref.
- **An unrecognized command may run as a shell command.** On older builds
  dispatch is a bare `$COMMAND "$@"`, so `git release ls` runs `ls` and
  `git release echo hi` prints `hi` — never pass an untrusted string as the
  subcommand. Guarded builds exit 64 with a usage error. Verify command names
  against `git release help` either way.
- **`readin` and `edit` hard-reset the tree.** `readinreleasefile` ends in
  `git reset --hard`; `edit` opens `nano` first. Both are human-only.
- **Destructive, confirm with the user first:** `dump` (deletes the RC locally
  and on origin, decrements the candidate — and requires a literal uppercase
  `Y`), `cleanrelease` / `cleanup` (delete old release branches on origin),
  `cleanupmergedremotebranches` (deletes remote branches), and
  `purgelocalbranches` (sweeps every local branch except main with no prompt at
  all — it uses `git branch -d`, so unmerged branches are refused, but nothing
  else is).
- **`auto` is a sticky mass-delete answer** in `cleanupmergedlocalbranches` and
  `cleanupmergedremotebranches`. It is not "yes to this one": `CHOICE` persists
  across iterations, so one `auto` deletes every remaining branch with no
  further prompt. Guarded builds say so in the prompt text.
- **Older builds delete release branches they meant to protect.**
  `cleanupmergedremotebranches` compares `remotes/origin/<name>` against
  `releases.branches`, which stores `origin/<name>`, so the "Found in current
  release" check never matches and branches in the active release are offered
  up like any other. Verified. Check `git release branches` against the
  deletion list yourself before confirming anything.
- **Never report a state you did not read back.** These commands are chatty and
  keep going after a failed step; a wall of output is not proof. Re-run
  `git release status` or the specific `git` query and quote what it said.

## 8. Commands that need a terminal, and what they do without one

**First, know which build you are on.** Recent builds guard every prompt; older
ones do not, and the difference decides whether a mistake is a clean failure or
silent corruption.

```bash
git release help | grep -q '\[tty\]' && echo "guarded" || echo "UNGUARDED — read the table"
```

These commands ask a question, and are human-only either way:

`init` (without both args) · `checkout` (without `true`) · `feature` · `rm` ·
`checkoutfeature` · `newfeature` · `pushfeature` · `devfeature` · `duplicate` ·
`dump` · `edit` · `cleanup` · `cleanrelease` · `cleanupmergedlocalbranches` ·
`cleanupmergedremotebranches` · `updatelocal` · `createdevrelease` ·
`deploy` (without an environment) · `set*deployurl` (without a value)

**On a guarded build** they print an error and **exit 78** having done nothing.
Attempting one costs you an exit code, not state.

**On an unguarded build** the outcome depends on stdin, and neither branch is
safe:

| stdin | What `read` does | Result |
|---|---|---|
| open — a pipe, a PTY nobody types into | blocks forever | the command **hangs** until killed |
| at EOF — `< /dev/null`, a closed pipe | returns non-zero, leaves the variable **empty** | the command **continues as if empty were the answer** |

The EOF case is the one that bites, because it looks like success — exit 0, no
error. Confirmed consequences on an unguarded build:

- `git release init` writes `releases.version=""` and
  `releases.current=release-v`. A later `roll` builds on that.
- `git release deploy` leaves the environment unresolved, so `git checkout ""`
  fails, execution continues, and `git reset --hard <main>` **rewrites whichever
  branch you were standing on**, destroying committed work there. Recoverable
  only via reflog.
- `git release add` (no ref) appends an empty entry to `releases.branches`.

y/n confirmations happen to fail safe on EOF (`dump`, the cleanup prompts read
empty and decline), but do not rely on that — it is a coincidence of each call
site, not a design.

**Never pipe answers in to get past a prompt.** `printf 'y\nauto\n' | git
release cleanupmergedremotebranches` is not "answering the first two questions":
`auto` is a sticky mode that deletes **every remaining branch** with no further
prompt, because `CHOICE` persists across loop iterations. On a guarded build
that pipe is refused outright; the deliberate opt-in is
`GIT_RELEASE_ASSUME_TTY=1`, which you should not reach for on a user's behalf.

### Safe unattended

On **any** build:

`status`, `version`, `candidate`, `current`, `branches`, `releasebranch`,
`nextreleasebranch`, `add <ref>`, `remove <ref>`, `roll`, `next`, `append`,
`push`, `to <target>`, `tag`, `setcandidate <n>`, `checkout true`,
`init <version> <candidate>`, and the `set*branch` commands *with* their value.

On a **guarded** build additionally — these have a prompt, but it is an optional
follow-up offer that arrives *after* the real work succeeded, so it is declined
with a `NOTICE` and the command still exits 0:

`stage` and `qa` (the deploy-webhook offer), `merge <branch>` (the tag offer
when the branch is main), `deploy <env>` (with the environment given).

On an unguarded build those four are **not** safe: `merge main` prompts for the
tag, and `stage`/`qa` prompt whenever a deploy URL is configured.

`checkout true` is non-interactive but not inert: it switches to the newest RC
for the current release, rewrites `releases.*`, clears and re-reads the branch
list, and hard-resets the tree.

## 9. Verify before "done"

- After a cut: `git release status`, and confirm the branch exists on origin —
  `git ls-remote --heads origin "$(git release releasebranch)"`. The push is
  skipped on conflicts, so a local branch proves nothing.
- After `add`/`remove`: `git release branches`.
- After a candidate-number fix: `git release candidate`.
- Anything you did not read back is a **carried** claim; say so.

For the full command tree see `references/command-map.md`.
