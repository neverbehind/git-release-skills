# `git-release` command map

Regenerate from `git release help`; this is a reading aid, not the source of
truth — and the built-in help is wrong about `reset` and `rm`. **[P]** marks a
command that blocks on `read`: human-only, never in an unattended session.

| Area | Commands | Touches |
|---|---|---|
| Read state | `status`, `deploystatus`, `version`, `candidate`, `current`, `branches`, `releasebranch`, `nextreleasebranch` | nothing — except `deploystatus`, which checks out and pulls the RC |
| Initialize | `init [version] [candidate]` (**[P]** without both), `duplicate` **[P]**, `checkout [true]` (**[P]** without `true`) | `releases.version` / `.candidate` / `.current`; clears `releases.branches` |
| Branch list | `add <ref>`, `remove <ref>`, `feature <search>` **[P]**, `rm` **[P]** (takes no argument) | `releases.branches` (multi-value); no dedupe on `add` |
| Cut a candidate | `roll`, `next`, `append`, `push` | new/updated RC branch; commits `releases/<current>` + `version`; runs `afterversioncommit.sh`; bumps `releases.candidate` (`roll`/`next` only) |
| Deploy | `to <target>`, `stage` (**[P]** with a webhook), `qa` (**[P]** with a webhook), `deploy [env]`, `merge <branch>`, `tag` | rewrites and force-pushes `<target>`; `merge` is the safe merge-back; see the `git-release-deploy` skill |
| Inspect environments | `releasebranches`, `devbranches` (**[P]** if `devbranch` unset), `stagebranches`, `qabranches` | checks out and pulls that branch; compares against **local** main |
| Configure | `setmainbranch`, `setstagebranch`, `setqabranch`, `setdevelopbranch`, `setcandidate <n>`, `setdevdeployurl`, `setstagedeployurl`, `setqadeployurl` (**[P]** without a value) | `releases.*` in `.git/config` |
| Release file | `writeout`, `readin` **[P]**, `edit` **[P]** | `releases/<current>`; `readin`/`edit` end in `git reset --hard` |
| Feature helpers | `newfeature` **[P]**, `checkoutfeature` **[P]**, `pushfeature` **[P]**, `devfeature <branch>` **[P]**, `updatelocal` **[P]** | ordinary branch/push/merge work; see `git-release-setup` §5 |
| Destructive | `dump` **[P]** (uppercase `Y`), `cleanup` **[P]**, `cleanrelease` **[P]**, `cleanupmergedlocalbranches` **[P]**, `cleanupmergedremotebranches` **[P]**, `purgelocalbranches` (no prompt; `branch -d`), `reset` | delete branches locally and on origin; `reset` is `git reset --hard`, *not* a config reset |
| Maintenance | `upgrade`, `help` | overwrites `~/bin/git-release` |
| Experimental | `createdevrelease` **[P]**, `movedevbranch` | `movedevbranch` hard-resets the develop branch onto the RC |

Useful shapes:

```bash
git release status
git config --local --get-regexp '^releases\.'          # raw state, no side effects

git release init 1.4.0 0 && git release setmainbranch main
git fetch --all && git branch -r | grep -i login       # resolve the ref yourself
git release add origin/feature/login
git release branches

git release roll                                        # first RC, or after a removal
git release next                                        # carry the RC forward
git release append                                      # re-merge in place

git diff --name-only --diff-filter=U                    # what halted the push
git commit -a -m "resolved conflicts" && git release push

git release to dev                                      # rebuild dev from origin/main + RC
git release merge main && git release tag               # merge-back, then v$(version)

REL=$(git release releasebranch)
git ls-remote --heads origin "$REL"                     # did the cut actually push?
git fetch --all && git merge-base --is-ancestor "$REL" origin/main && echo shipped
git release setcandidate 3                              # after a failed roll
```
