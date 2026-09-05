---
name: git-release-deploy
description: >-
  Use when moving a `git-release` candidate toward or into production —
  `git release to <target>`, `stage`, `qa`, `deploy`, `merge`, `tag`, deploy
  webhooks, and merge-back to main. Covers the contract that makes deploy
  trigger branches (dev/stage/qa) disposable and force-pushed, why merge-back
  to main happens after the deploy and never before, the guards that were
  tried here and broke the workflow, and how to verify what is actually
  running in an environment.
user-invocable: true
allowed-tools: Bash, Read, Grep, Glob
argument-hint: "[environment or target branch you are deploying to]"
---

# git-release-deploy — shipping a candidate

Deployment in `git-release` is "put the RC on a branch a build system watches."
There are two generations of command for that, and one contract that explains
the whole design.

Cutting candidates and managing the branch list live in the `git-release` skill.

## 1. The contract: the deploy target is disposable

```bash
git release to dev          # or stage / qa / any non-main branch
```

`to <target>` does exactly this:

```
fetch --all
checkout <target>
reset --hard origin/<mainbranch>      # <- the whole point
merge --no-ff <release branch>
push origin <target> -f
```

`<target>` is **rebuilt from scratch on every deploy** as
`origin/main + release branch`, and force-pushed. Deploy-trigger branches carry
no history of their own and are never a source of truth. Anything committed
directly onto `dev`/`stage`/`qa` is discarded on the next deploy — by design.
The `-f` is the tell.

Consequences you must not fight:

- **Never guard `<target>`'s prior state.** A check that stops the push because
  local `<target>` is behind `origin/<target>` is defending the very thing this
  command exists to rewind. It deadlocks the second deploy of any RC.
- **The reset base is `origin/<mainbranch>`, not local `main`.** `fetchall` has
  just refreshed the remote ref; the operator's local main may be arbitrarily
  stale and would silently deploy old code.
- `to` refuses `<target>` == the main branch. Use `git release merge <main>`
  for that; see §4.

## 2. Merge-back happens after the deploy

The workflow order is: cut RC → deploy to an environment → test → merge back to
main → tag. So at deploy time, `origin/main` **does not** contain the release
yet, and it is not supposed to.

`to` therefore asserts nothing about main after the push. A post-push "is the
release an ancestor of `origin/main`?" check fails on every single deploy and
returns non-zero even though the push succeeded. The place to catch a
never-merged-back release is when the *next* release is cut from main, not here.

**`Already up to date.` from the release merge is a NOTICE, not an error.**
Since `<target>` was just reset to `origin/main`, a no-op merge means
`origin/main` already contains the release tip — normal when a previous cycle's
merge-back landed, or when redeploying an already-merged RC. The tool prints the
notice and continues; `<target> == origin/main` is the correct deploy state.

### Regression history — do not reintroduce

Commit `531692e` in `neverbehind/git-release` shipped three checks in `to` that
each broke normal use. All were removed. If you are reading a build between
`531692e` and its revert, upgrade (`git release upgrade`).

| Check | What it did | Why it broke |
|---|---|---|
| reset base → `origin/<target>` | preserved the target's history | defeated the rebuild-from-main contract entirely; the deploy branch was never reset |
| **R-e** | aborted when local `<target>` was behind `origin/<target>` | defended the branch the command exists to force-push; deadlocked the second deploy of an RC |
| **R-a** + **R-f** | required the release in `origin/main` after the push, *and* hard-failed on `Already up to date.` | mutually contradictory: R-a demanded merge-back, R-f punished the topology a healthy merge-back produces. R-a also fired on every deploy |

`GIT_RELEASE_SKIP_ANCESTOR_CHECK` was R-a's escape hatch. The script no longer
reads it — drop it from any wrapper.

## 3. The older per-environment commands

```bash
git release stage      # merge RC into $(stagebranch), push, offer the webhook
git release qa         # merge RC into $(qabranch),   push, offer the webhook
git release deploy [dev|stage|qa|prod]
```

- `stage` and `qa` **merge onto** the target (checkout + `git pull` + merge +
  plain push). They accumulate history rather than rebuilding, so an environment
  branch drifts from main over time. `to <target>` is the current mechanism.
- **`stage` and `qa` ask before `curl`-ing the webhook** when
  `releases.stagedeployurl` / `releases.qadeployurl` is set. The merge and push
  have already happened by then, so guarded builds decline the offer with a
  `NOTICE` and still exit 0 — safe unattended, the webhook just does not fire.
  On older builds the same prompt hangs (or, at EOF, silently declines). Either
  run them with a human present, or use `to <target>` and fire the webhook
  yourself.
- **`deploy` needs its environment as an argument.** Without one it asks, and on
  an unguarded build an empty answer is catastrophic: `TRUNK_BRANCH` stays
  empty, `git checkout ""` fails, execution continues, and
  `git reset --hard <main>` **rewrites whatever branch you were standing on**,
  destroying committed work there. Verified. Guarded builds exit 78 (no
  environment) or 1 (unknown environment) without touching the tree.
- `deploy <env>` maps the env to a configured branch, then for non-prod does
  `git reset --hard "$(mainbranch)"` — same rebuild intent as `to`, but off
  **local** main. That is the staleness hazard `to` was changed to avoid; it is
  a known parity gap. Prefer `to` for dev/stage/qa.
- `deploy prod` is different: it checks out main, `git pull`s (no reset), merges
  the RC, force-pushes, and tags. Force-pushing main is a real risk — confirm
  with the user before running it, and prefer the `merge` path in §4.

## 4. Merge-back to main, and tagging

```bash
git release merge main     # checkout, pull, merge --no-ff RC, plain push
                           # then offers to tag when the target is the main branch
git release tag            # git tag v$(version) at HEAD; git push --tags
```

`merge` is the safe merge-back: a normal push, no reset, no force. Use it rather
than `deploy prod`.

**`merge <main>` asks whether to tag.** The merge and push are already done when
it asks, so a guarded build declines and exits 0, leaving you to run
`git release tag` explicitly — which is what you want anyway, since `tag` needs
you to confirm your position first. On an older build that prompt hangs an
unattended session; `merge <branch>` is only unconditionally safe when the
branch is *not* the main branch.

`tag` tags **whatever is currently checked out** with `v$(version)` and pushes
all tags. It does not verify you are on main and does not create an annotated
tag. Confirm your position first:

```bash
git symbolic-ref --short HEAD          # must be the main branch
git release version                    # the tag that will be created
git release tag
```

The version comes from `releases.version`, so the tag is `v1.4.0` regardless of
which RC produced it.

## 5. Webhooks

```bash
git release setdevdeployurl   <url>    # called after a dev merge
git release setstagedeployurl <url>
git release setqadeployurl    <url>
```

Stored in `.git/config` as `releases.*deployurl`; pass an empty value to unset.
They are fired by `stage`, `qa`, and `devfeature` — behind a y/n prompt — and
never by `to`. If your pipeline is triggered by the push itself, leave them
unset; `to` is then fully non-interactive. Note that an unattended `stage`/`qa`
on a guarded build **declines** the offer: the branch is deployed but the
webhook never fires, and the command still exits 0. If the webhook is what
actually triggers your build, fire it yourself:

```bash
git release stage && curl -fsS "$(git config --get releases.stagedeployurl)"
```

## 6. Verify what is actually deployed

The push output is not proof. Read the remote back:

```bash
git fetch --all
REL=$(git release releasebranch)
git rev-parse origin/dev                                   # what dev points at
git merge-base --is-ancestor "$REL" origin/dev && echo "dev has $REL"
git merge-base --is-ancestor "$REL" origin/main && echo "merged back to main"
git ls-remote --tags origin | grep "v$(git release version)"
```

`git release status` reports Pending/Deployed by comparing against **local**
main, so it can claim "Pending" for a release that shipped. Fetch first, or use
the `merge-base` check above.

## 7. Guardrails

- Confirm with the user before anything that force-pushes a branch a human might
  be committing to, and before `deploy prod` in any form.
- Always pass `deploy` its environment. A bare `git release deploy` is the one
  command here that can destroy local work outright — see §3.
- `to`, `stage`, `qa`, `deploy`, and `merge` all leave you standing on the target
  branch. Record the starting branch and check it back out.
- A `to` run over a dirty tree loses that work to `reset --hard`. Check
  `git status --porcelain` first.
- Deploy the RC that exists on origin, not just locally: a cut halted by
  conflicts is never pushed, so `to` would deploy a branch the build system
  cannot fetch. Confirm with
  `git ls-remote --heads origin "$(git release releasebranch)"`.
- Reproducible behavior tests for `to` live in `scenarios/` in the git-release
  repo (`scenario_b_target_rebuilt_from_main.sh`,
  `scenario_c_no_op_merge_notice.sh`). They build throwaway sandbox repos —
  never point them at a working clone.
