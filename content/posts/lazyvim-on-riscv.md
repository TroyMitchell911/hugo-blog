+++
date = '2026-04-20T17:09:46+08:00'
draft = false
title = 'Building LazyVim on RISC-V from Scratch'
tags = ["riscv", "neovim"]
summary = 'A complete guide to compiling and running Neovim with LazyVim on RISC-V Linux, including LuaJIT porting, tree-sitter, LSP, blink.cmp native build, and snacks.nvim workarounds.'
+++

## Background

LazyVim is a Neovim distribution built on lazy.nvim that provides a modern, batteries-included editor experience. However, lazy.nvim requires Neovim to be built with LuaJIT, and upstream LuaJIT does not support RISC-V.

This post documents the full process of building and running Neovim + LazyVim on a RISC-V Linux environment (Ubuntu/Debian), including all the pitfalls and solutions.

Test environment:
- Architecture: riscv64gc
- OS: Ubuntu (bianbu/resolute)
- Board: K3 development board

---

## 1. Building RISC-V LuaJIT

Upstream LuaJIT doesn't support RISC-V, but there's an actively maintained community port: [IgnotaYun/LuaJIT](https://github.com/IgnotaYun/LuaJIT) (`riscv` branch). The project has green CI and is production-ready.

### 1.1 Install build dependencies

```bash
sudo apt install -y \
  build-essential \
  git
```

### 1.2 Build LuaJIT

```bash
git clone -b riscv https://github.com/IgnotaYun/LuaJIT.git ~/luajit-riscv
cd ~/luajit-riscv
make -j$(nproc)
sudo make install
```

### 1.3 Configure dynamic linker

If LuaJIT was installed to `/usr/local/lib`, make sure the system linker can find it:

```bash
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/luajit.conf
sudo ldconfig
```

### 1.4 Verify

```bash
luajit -v
# Should output something like: LuaJIT 2.1.xxxxxxxxxx -- Copyright (C) 2005-2025 Mike Pall. https://luajit.org/
```

---

## 2. Building Neovim

### 2.1 Install build dependencies

```bash
sudo apt install -y \
  ninja-build \
  gettext \
  libtool \
  libtool-bin \
  autoconf \
  automake \
  cmake \
  g++ \
  pkg-config \
  unzip \
  curl \
  git
```

### 2.2 Get the source

```bash
git clone https://github.com/neovim/neovim.git ~/neovim
cd ~/neovim
git checkout stable  # or a specific tag like v0.12.1
```

### 2.3 Build third-party dependencies

The key here is `-DUSE_BUNDLED_LUAJIT=OFF` — this tells Neovim to skip downloading its bundled LuaJIT and use the system-installed RISC-V LuaJIT instead.

```bash
cmake -S cmake.deps -B .deps \
  -DUSE_BUNDLED_LUAJIT=OFF

cmake --build .deps -j$(nproc)
```

### 2.4 Build Neovim

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release

cmake --build build -j$(nproc)
```

If cmake can't find the system LuaJIT, specify the paths manually:

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DLUAJIT_INCLUDE_DIR=/usr/local/include/luajit-2.1 \
  -DLUAJIT_LIBRARY=/usr/local/lib/libluajit-5.1.so
```

You can locate the paths with:

```bash
find /usr -name "luajit.h" 2>/dev/null
find /usr -name "libluajit*" 2>/dev/null
```

### 2.5 Install

```bash
sudo cmake --install build
```

### 2.6 Verify

```bash
nvim --version
```

The output should contain `LuaJIT 2.1.xxx`, confirming Neovim is correctly linked against the RISC-V LuaJIT.

---

## 3. Installing LazyVim

See the official docs: https://www.lazyvim.org/installation

### 3.1 Back up existing config (if any)

```bash
mv ~/.config/nvim{,.bak}
mv ~/.local/share/nvim{,.bak}
mv ~/.local/state/nvim{,.bak}
mv ~/.cache/nvim{,.bak}
```

### 3.2 Clone LazyVim Starter

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
```

### 3.3 Launch Neovim

```bash
nvim
```

On first launch, lazy.nvim will automatically download and install all plugins.

---

## 4. Fixing tree-sitter-cli

### 4.1 Problem

LazyVim uses tree-sitter for syntax highlighting. The `tree-sitter-cli` npm package has no prebuilt binary for RISC-V, so `:TSInstall` fails.

### 4.2 Solution: Install via Cargo

```bash
cargo install tree-sitter-cli
```

If `cargo` is not installed:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

> **Note:** On RISC-V, `BINDGEN_EXTRA_CLANG_ARGS` may need to be set for some crates that use bindgen:
> ```bash
> export BINDGEN_EXTRA_CLANG_ARGS="--target=riscv64-linux-gnu -I/usr/include -I/usr/include/riscv64-linux-gnu"
> ```

---

## 5. Fixing lua_ls (Lua Language Server)

### 5.1 Problem

Mason can't install `lua-language-server` on RISC-V because there's no prebuilt binary.

### 5.2 Solution: Build from source

```bash
sudo apt install -y ninja-build
git clone --depth=1 https://github.com/LuaLS/lua-language-server ~/lua-language-server
cd ~/lua-language-server
./make.sh
```

Add to PATH:

```bash
echo 'export PATH="$HOME/lua-language-server/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 5.3 Tell LazyVim to skip Mason for lua_ls

```lua
-- ~/.config/nvim/lua/plugins/lsp.lua
return {
  {
    "neovim/nvim-lspconfig",
    opts = {
      servers = {
        lua_ls = {
          mason = false,
        },
      },
    },
  },
}
```

---

## 6. Fixing blink.cmp

### 6.1 Problem

On startup, nvim shows:

```
blink.cmp  ENOENT: no such file or directory: ...
blink.cmp  Falling back to Lua implementation due to error while downloading pre-built binary
```

### 6.2 Cause

blink.cmp is LazyVim's default completion plugin. It includes a high-performance fuzzy matching library written in Rust. On startup, it tries to download a prebuilt `.so` file, but there's no RISC-V build available, so it falls back to a pure Lua implementation.

### 6.3 Solution: Build the native library locally

Enter the plugin directory:

```bash
cd ~/.local/share/nvim/lazy/blink.cmp
```

Build:

```bash
cargo build --release
```

The output is at `target/release/libblink_cmp_fuzzy.so`. blink.cmp's Lua loader automatically searches `<plugin-root>/target/release/` for this file — no manual copying needed.

Verify:

```bash
ls target/release/libblink_cmp_fuzzy.so
```

If the error persists, disable prebuilt downloads in the config (optional):

```lua
-- ~/.config/nvim/lua/plugins/blink.lua
return {
  {
    "saghen/blink.cmp",
    opts = {
      fuzzy = {
        prebuilt_binaries = {
          download = false,
        },
      },
    },
  },
}
```

Restart nvim — blink.cmp will now use the locally compiled native Rust library.

> **Note:** After blink.cmp plugin updates, you may need to re-run `cargo build --release` since the Rust source may have changed.

---

## 7. Fixing snacks.nvim indent/scope Error

### 7.1 Problem

Opening a file in nvim triggers:

```
Error in BufReadPost Autocommands for "*":
Lua callback: ...snacks.nvim/lua/snacks/scope.lua:136: attempt to index local 'self' (a nil value)
```

### 7.2 Cause

It's unclear whether this is a snacks.nvim bug or a RISC-V LuaJIT compatibility issue.

The likely cause: `snacks/scope.lua` line 136 uses the `__eq` metamethod to compare two scope objects. Per the Lua spec, `__eq` should only be invoked when both operands are the same type and both have `__eq`. However, the IgnotaYun RISC-V LuaJIT branch may have a subtle behavioral difference in the `__eq` JIT compilation path, causing `nil` values to unexpectedly enter the metamethod call, triggering the error.

### 7.3 Solution: Disable snacks.nvim indent feature

```lua
-- ~/.config/nvim/lua/plugins/snacks.lua
return {
  {
    "folke/snacks.nvim",
    opts = {
      indent = { enabled = false },
    },
  },
}
```

Impact: You lose indent guide lines and current scope highlighting. This is purely visual — it doesn't affect editing, LSP, completion, treesitter, or any core functionality.

---

## 8. Summary

| Component | Problem | Solution |
|-----------|---------|----------|
| LuaJIT | No RISC-V support upstream | Use IgnotaYun/LuaJIT `riscv` branch |
| Neovim | Bundled LuaJIT doesn't support RV | `-DUSE_BUNDLED_LUAJIT=OFF` to use system LuaJIT |
| tree-sitter-cli | No prebuilt binary | `cargo install tree-sitter-cli` |
| lua_ls | No Mason prebuilt binary | Build from source + `mason = false` |
| blink.cmp | No prebuilt .so | `cargo build --release` in plugin directory |
| snacks.nvim | indent/scope triggers LuaJIT `__eq` anomaly | `indent = { enabled = false }` |

### Recommended shell environment variables

```bash
# ~/.zshrc or ~/.bashrc
export BINDGEN_EXTRA_CLANG_ARGS="--target=riscv64-linux-gnu -I/usr/include -I/usr/include/riscv64-linux-gnu"
export PATH="$HOME/lua-language-server/bin:$PATH"
```

After completing all the steps above, you'll have a fully functional LazyVim development environment on RISC-V.
