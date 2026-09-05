# `git-release` command map

Regenerate from `git release help`; this is a reading aid, not the source of
truth — and older built-in help is wrong about `reset` and `rm`.

**[P]** marks a command that asks a question: human-only. On a guarded build
(`git release help | grep -q '\[tty\]'`) it exits 78 having done nothing. On an
older build it hangs when stdin is open, or acts on an **empty answer** when
stdin is at EOF — see §8 of the skill. **[P?]** marks a prompt that arrives only
after the command's real work succeeded; guarded builds decline it and exit 0,
older builds block.

| Area | Commands | Touches |
|---|---|---|
| Read state | `status`, `deploystatus`, `version`, `candidate`, `current`, `branches`, `releasebranch`, `nextreleasebranch` | nothing — except `deploystatus`, which checks out and pulls the RC |
| Initialize | `init [version] [candidate]` (**[P]** without both), `duplicate` **[P]**, `checkout [true]` (**[P]** without `true`) | `releases.version` / `.candidate` / `.current`; clears `releases.branches` |
| Branch list | `add <ref>`, `remove <ref>`, `feature <search>` **[P]**, `rm` **[P]** (takes no argument) | `releases.branches` (multi-value); no dedupe on `add`; a bare `add` stores an empty entry on older builds |
| Cut a candidate | `roll`, `next`, `append`, `push` | new/updated RC branch; commits `releases/<current>` + `version`; runs `afterversioncommit.sh`; bumps `releases.candidate` (`roll`/`next` only) |
| Deploy | `to <target>`, `stage` (**[P?]** with a webhook), `qa` (**[P?]** with a webhook), `deploy <env>` (**[P]** without the env — destroys the current branch on older builds), `merge <branch>` (**[P?]** when the branch is main), `tag` | rewrites and force-pushes `<target>`; `merge` is the safe merge-back; see the `git-release-deploy` skill |
| Inspect environments | `releasebranches`, `devbranches` (**[P]** — `devbranch` has no default), `stagebranches`, `qabranches` (their prompts are unreachable; those keys default) | checks out and pulls that branch; compares against **local** main |
| Configure | `setmainbranch`, `setstagebranch`, `setqabranch`, `setdevelopbranch`, `setcandidate <n>`, `setdevdeployurl`, `setstagedeployurl`, `setqadeployurl` (**[P]** without a value) | `releases.*` in `.git/config` |
| Release file | `writeout`, `readin` **[P]**, `edit` **[P]** | `releases/<current>`; `readin`/`edit` end in `git reset --hard` |
| Feature helpers | `newfeature` **[P]**, `checkoutfeature` **[P]**, `pushfeature` **[P]**, `devfeature <branch>` **[P]**, `updatelocal` **[P]** | ordinary branch/push/merge work; see `git-release-setup` §5 |
| Cut a candidate — caveats | `roll`, `next`, `append` | no dirty-tree check; exit 1 on conflict and skip the push, but exit **0 even if the push fails** |
| Destructive | `dump` **[P]** (uppercase `Y`), `cleanup` **[P]**, `cleanrelease` **[P]**, `cleanupmergedlocalbranches` **[P]**, `cleanupmergedremotebranches` **[P]**, `purgelocalbranches` (no prompt; `branch -d`), `reset` | delete branches locally and on origin; `reset` is `git reset --hard`, *not* a config reset; answering `auto` in either `cleanupmerged*` deletes **all remaining** branches unprompted |
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

git release help | grep -q '\[tty\]' && echo guarded    # can this build refuse to prompt?
```

Exit codes on a guarded build: **78** = needs an answer typed at a terminal;
**64** = not a git release command; **1** = the command's own failure.
