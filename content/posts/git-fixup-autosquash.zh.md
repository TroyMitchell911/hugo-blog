+++
date = '2026-04-28T09:32:38+08:00'
draft = false
title = '别再手动编辑 rebase TODO 了：用 git commit --fixup + autosquash'
tags = ['git']
summary = '使用 git commit --fixup 和 rebase --autosquash 更快地修改 patch series 中的历史 commit，不用手动编辑 rebase TODO list。'
+++

如果你维护过 patch series（比如内核 upstream），你大概做过无数次这样的操作：

```
git rebase -i <base>
# 把 "pick" 改成 "edit"
# 改代码，git add，git commit --amend
# git rebase --continue
```

能用，但很烦——尤其是要同时修多个不同的 commit 时。有更好的办法。

## 工作流

### 1. 改完代码，用 `--fixup`

改完文件后，不要急着 rebase：

```bash
git add -u
git commit --fixup=<target-commit>
```

这会创建一个带特殊前缀的 commit：

```
fixup! <原始 commit message>
```

它就是个普通 commit，只是名字有标记。你可以连续创建多个 fixup commit，分别指向不同的目标 commit，最后一次性 rebase。

### 2. 一次 autosquash 全部合入

```bash
git rebase -i --autosquash <base>
```

Git 看到 `fixup!` 前缀，会自动重排 TODO list，把 fixup commit 移到目标 commit 下面并标记为 `fixup`。你什么都不用改——直接保存退出编辑器。

如果你信任自动化，不想看 TODO：

```bash
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <base>
```

`GIT_SEQUENCE_EDITOR=true` 直接跳过编辑器。

## 用 `:/` 省掉查 hash 的步骤

`--fixup=<hash>` 最烦的是要先查 hash。可以用正则搜索代替：

```bash
git commit --fixup=:/keyword
```

`:/` 告诉 git 搜索 commit message，找到最近一条匹配的 commit。例如：

```bash
git commit --fixup=:/fixed-sample-rate
git commit --fixup=:/dt-bindings.*K3
git commit --fixup=:/FIFO trigger
```

只要能唯一匹配就行。匹配不到或匹配多条，git 会报错——不会搞错。

## 实际例子

假设你有一个 7-patch series，发现 patch 4（`ASoC: dt-bindings: add SpacemiT K3 SoC compatible`）和 patch 7（`ASoC: spacemit: add K3 SoC support with additional clocks`）有 bug。

修 YAML 文件：
```bash
git add Documentation/devicetree/bindings/sound/spacemit,k1-i2s.yaml
git commit --fixup=:/dt-bindings.*K3
```

修 C 文件：
```bash
git add sound/soc/spacemit/k1_i2s.c
git commit --fixup=:/K3 SoC support
```

现在 log 长这样：
```
fixup! ASoC: spacemit: add K3 SoC support with additional clocks
fixup! ASoC: dt-bindings: add SpacemiT K3 SoC compatible
ASoC: spacemit: add K3 SoC support with additional clocks
ASoC: spacemit: add fixed-sample-rate constraint support
ASoC: dt-bindings: add fixed-sample-rate property for SpacemiT K1/K3
ASoC: dt-bindings: add SpacemiT K3 SoC compatible
...
```

一条命令全部合入：
```bash
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <base>~1
```

搞定。两个 fixup 分别合进了各自的目标 commit。历史干净，不用手动编辑。

## `--fixup` vs `--squash`

| | `--fixup` | `--squash` |
|---|---|---|
| 前缀 | `fixup!` | `squash!` |
| rebase 时 | 静默合并，保留原 message | 打开编辑器合并 message |
| 适用场景 | 修 bug、静默修正 | 补充 commit message |

维护 patch series 的话，`--fixup` 几乎总是你要的。

## 总结

| 老办法 | 新办法 |
|---|---|
| `git rebase -i`，手动改 `pick` → `edit` | `git commit --fixup=:/keyword` |
| 停在 commit，amend，continue | `git rebase -i --autosquash` |
| 一次只能修一个 | 批量 fixup，一次 rebase |
