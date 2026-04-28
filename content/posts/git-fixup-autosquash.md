+++
date = '2026-04-28T09:32:38+08:00'
draft = false
title = 'Stop Manually Editing Rebase TODOs: Use git commit --fixup + autosquash'
tags = ['git']
summary = 'A faster way to amend older commits in a patch series using git commit --fixup and rebase --autosquash, without manually editing the rebase TODO list.'
+++

If you maintain patch series (for kernel upstream, for example), you've probably done this dance many times:

```
git rebase -i <base>
# change "pick" to "edit" on the target commit
# make your fix, git add, git commit --amend
# git rebase --continue
```

It works, but it's tedious — especially when you have multiple fixes targeting different commits. There's a better way.

## The Workflow

### 1. Make your fix, then `--fixup`

After editing the file, instead of starting a rebase right away:

```bash
git add -u
git commit --fixup=<target-commit>
```

This creates a commit with a special message prefix:

```
fixup! <original commit message>
```

It's just a normal commit with a marker. You can create multiple fixup commits targeting different commits before rebasing.

### 2. Autosquash them all at once

```bash
git rebase -i --autosquash <base>
```

Git sees the `fixup!` prefixes, automatically reorders the TODO list, and marks them as `fixup`. You don't touch anything — just save and quit the editor.

If you trust the automation and don't need to review the TODO:

```bash
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <base>
```

`GIT_SEQUENCE_EDITOR=true` skips the editor entirely.

## Skip the Hash Lookup with `:/`

The annoying part of `--fixup=<hash>` is finding the hash. You can use a regex search instead:

```bash
git commit --fixup=:/keyword
```

`:/` tells git to search commit messages. It finds the most recent commit whose message matches. Examples:

```bash
git commit --fixup=:/fixed-sample-rate
git commit --fixup=:/dt-bindings.*K3
git commit --fixup=:/FIFO trigger
```

As long as the match is unambiguous, it works. If nothing matches or multiple commits match, git errors out — no silent mistakes.

## A Real Example

Say you have a 7-patch series and discover a bug in patch 4 (`ASoC: dt-bindings: add SpacemiT K3 SoC compatible`) and patch 7 (`ASoC: spacemit: add K3 SoC support with additional clocks`).

Fix the YAML file:
```bash
git add Documentation/devicetree/bindings/sound/spacemit,k1-i2s.yaml
git commit --fixup=:/dt-bindings.*K3
```

Fix the C file:
```bash
git add sound/soc/spacemit/k1_i2s.c
git commit --fixup=:/K3 SoC support
```

Now your log looks like:
```
fixup! ASoC: spacemit: add K3 SoC support with additional clocks
fixup! ASoC: dt-bindings: add SpacemiT K3 SoC compatible
ASoC: spacemit: add K3 SoC support with additional clocks
ASoC: spacemit: add fixed-sample-rate constraint support
ASoC: dt-bindings: add fixed-sample-rate property for SpacemiT K1/K3
ASoC: dt-bindings: add SpacemiT K3 SoC compatible
...
```

One command to fold them in:
```bash
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <base>~1
```

Done. Both fixups are absorbed into their target commits. Clean history, no manual editing.

## `--fixup` vs `--squash`

| | `--fixup` | `--squash` |
|---|---|---|
| Prefix | `fixup!` | `squash!` |
| During rebase | Silently merges, keeps original message | Opens editor to combine messages |
| Use case | Bug fixes, silent corrections | Adding context to commit messages |

For patch series maintenance, `--fixup` is almost always what you want.

## Summary

| Old way | New way |
|---|---|
| `git rebase -i`, manually change `pick` → `edit` | `git commit --fixup=:/keyword` |
| Stop at commit, amend, continue | `git rebase -i --autosquash` |
| One fix at a time | Batch multiple fixups, rebase once |
