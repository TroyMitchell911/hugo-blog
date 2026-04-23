+++
date = '2024-11-30T12:07:41+08:00'
draft = false
title = 'Fix LazyVim Clipboard over SSH'
tags = ["neovim"]
summary = 'LazyVim disables system clipboard when running over SSH. A one-line config override fixes it.'
+++

## The Problem

You yank text in Neovim over SSH, switch to your local machine, and paste — nothing. The clipboard is empty.

Neovim (0.10+) has built-in clipboard support via the yank autocommand, so in a local session everything works fine. But LazyVim overrides this with a conditional in its default options:

```lua
opt.clipboard = vim.env.SSH_TTY and "" or "unnamedplus"
```

This checks whether the `SSH_TTY` environment variable is set. If it is (i.e., you're in an SSH session), LazyVim sets `clipboard` to an empty string, effectively disabling system clipboard sync. The rationale is that clipboard forwarding over SSH is unreliable in many setups, so LazyVim plays it safe.

## The Fix

Override this default in `~/.config/nvim/lua/config/options.lua`:

```lua
vim.opt.clipboard = "unnamedplus"
```

This forces Neovim to always sync yanked text to the system clipboard, regardless of whether you're in an SSH session.

## Making It Actually Work over SSH

Setting `clipboard = "unnamedplus"` alone isn't enough — you also need a working clipboard forwarding mechanism between the remote and local machine. Common approaches:

- **OSC 52** — Modern terminals (kitty, WezTerm, Alacritty, Windows Terminal) support the OSC 52 escape sequence, which lets the remote terminal write directly to the local clipboard. Neovim 0.10+ detects and uses this automatically.
- **X11 forwarding** — `ssh -X` or `ssh -Y` forwards the X clipboard. Requires `xclip` or `xsel` on the remote.
- **Dedicated tools** — [lemonade](https://github.com/lemonade-command/lemonade), [clipper](https://github.com/wincent/clipper), etc.

For most modern setups, OSC 52 is the simplest — it just works with no extra configuration if your terminal supports it.
