+++
date = '2024-11-30T12:07:41+08:00'
draft = false
title = '修复 LazyVim 在 SSH 下无法复制到剪贴板的问题'
tags = ["neovim"]
summary = 'LazyVim 在 SSH 会话中默认禁用系统剪贴板同步，一行配置即可修复。'
+++

## 问题

通过 SSH 在 Neovim 里 yank 了文本，切回本地机器粘贴 — 什么都没有。

Neovim（0.10+）内置了剪贴板支持，本地使用一切正常。但 LazyVim 在默认配置中做了一个条件判断：

```lua
opt.clipboard = vim.env.SSH_TTY and "" or "unnamedplus"
```

这行代码检查 `SSH_TTY` 环境变量是否存在。如果存在（即处于 SSH 会话中），LazyVim 会将 `clipboard` 设为空字符串，等于禁用了系统剪贴板同步。原因是 SSH 下的剪贴板转发在很多环境中不可靠，LazyVim 选择了保守策略。

## 修复

在 `~/.config/nvim/lua/config/options.lua` 中覆盖这个默认值：

```lua
vim.opt.clipboard = "unnamedplus"
```

这样无论是否在 SSH 会话中，Neovim 都会将 yank 的内容同步到系统剪贴板。

## 让剪贴板在 SSH 下真正工作

仅设置 `clipboard = "unnamedplus"` 还不够 — 还需要远程和本地之间有可用的剪贴板转发机制。常见方案：

- **OSC 52** — 现代终端（kitty、WezTerm、Alacritty、Windows Terminal）支持 OSC 52 转义序列，允许远程终端直接写入本地剪贴板。Neovim 0.10+ 会自动检测并使用，无需额外配置。
- **X11 转发** — `ssh -X` 或 `ssh -Y` 转发 X 剪贴板。远程需要安装 `xclip` 或 `xsel`。
- **专用工具** — [lemonade](https://github.com/lemonade-command/lemonade)、[clipper](https://github.com/wincent/clipper) 等。

对于大多数现代环境，OSC 52 是最简单的方案 — 只要终端支持就能直接用，不需要任何额外配置。
