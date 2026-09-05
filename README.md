# git-release-skills

Official Claude Code skills for [git-release](https://github.com/neverbehind/git-release), the Git Flow-style release candidate CLI, packaged as a plugin marketplace.

## Install

```bash
claude plugin marketplace add neverbehind/git-release-skills
claude plugin install git-release@git-release-skills      # --scope project to pin it in a workspace
```

Or inside a session: `/plugin marketplace add neverbehind/git-release-skills`, then `/plugin install git-release@git-release-skills`.

The skills wrap the `git-release` CLI; they install no tools. Install git-release itself with:

```bash
curl -s -L https://raw.githubusercontent.com/neverbehind/git-release/main/install.sh | bash
```

## What you get

One plugin, `git-release`, with three skills (invoke as `/git-release:<skill>` or let them auto-trigger):

| Skill | Audience | Covers |
|---|---|---|
| `git-release` | anyone cutting a release from Claude Code, including agents | the `.git/config` state model, building the branch list, `roll` vs `next` vs `append`, halted merges, reading what shipped, and which commands block on `read` |
| `git-release-deploy` | whoever ships the candidate | the disposable deploy-trigger contract behind `to <target>`, why merge-back to main comes after the deploy, the guards that broke this workflow once, webhooks, and verifying an environment |
| `git-release-setup` | whoever installs and configures it | install and upgrade, `init`, the `releases.*` keys and their defaults, the `afterversioncommit.sh` hook contract, feature helpers, cleanup, and a troubleshooting map |

## Boundary

These skills carry **tool mechanics only** and should be usable by any git-release user unchanged. Repo-specific content — your trunk and environment branch names, your deploy webhooks, what your `afterversioncommit.sh` does, who approves a production merge — belongs in that repo's `CLAUDE.md`, not here. Do not add it.

## Develop

```bash
claude plugin validate .                                # marketplace manifest
claude plugin validate plugins/git-release --strict     # plugin + skills
```

Bump `version` in both `plugins/git-release/.claude-plugin/plugin.json` and the plugin entry in `.claude-plugin/marketplace.json` on every content change; installed copies update on `claude plugin marketplace update git-release-skills`.
