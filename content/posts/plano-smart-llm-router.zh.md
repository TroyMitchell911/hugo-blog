+++
date = '2026-04-27T16:59:39+08:00'
draft = false
title = 'Plano：带智能路由和故障转移的 LLM 网关'
tags = ['LLM', 'AI']
summary = '部署 Plano 作为本地 LLM 代理，支持跨渠道自动故障转移、按状态码重试策略和基于意图的智能路由——让你的请求始终通过正确的渠道到达最合适的模型。'
+++

## Plano 是什么？

[Plano](https://github.com/katanemo/plano) 是一个开源的本地 LLM 网关，位于你的应用和上游 LLM 供应商之间。它对外暴露 OpenAI 兼容 API，所以任何能对接 OpenAI 的工具都能直接对接 Plano，无需改代码。

Plano 原生就支持多模型路由和基本的供应商管理。在此基础上，我添加了以下补丁：

- **自动重试** — 遇到限流 (429) 和服务端错误 (5xx) 时自动重试，支持指数退避和 `Retry-After` header
- **跨渠道故障转移** — 当一个渠道失败时，Plano 自动尝试 fallback 列表中的下一个渠道
- **同模型多渠道** — 同一个底层模型（如 `claude-opus-4-6`）可以配置在多个渠道前缀下，各自使用独立的 API key
- **基于意图的智能路由** — Plano 分析 prompt 内容，自动路由到最适合的模型（代码生成 → Opus，简单问题 → Flash 等）

补丁版 release 在 [TroyMitchell911/plano](https://github.com/TroyMitchell911/plano/releases/tag/v0.4.20-retry-failover)。

## 安装

### Linux（原生模式）

#### 第一步：安装 CLI

CLI（`planoai`）是一个 Python 包，负责生成 Envoy 配置并管理 Plano 的生命周期。

**如果你的系统可以正常 `pip install`：**

```bash
pip install "planoai @ git+https://github.com/TroyMitchell911/plano.git@feat/retry-failover#subdirectory=cli"
```

> 必须从 fork 安装，不能从 PyPI 安装。PyPI 上的 `planoai==0.4.20` 不支持同模型多渠道配置，会报 `Duplicate model_id` 错误。

**如果你的系统锁定了 pip（externally-managed-environment）：**

很多现代 Linux 发行版（Arch、Fedora 38+、Ubuntu 23.04+、Debian 12+）使用 PEP 668 锁定了系统 Python，你会看到：

```
error: externally-managed-environment
```

两种解决方案：

**方案 A：pipx（推荐）**

```bash
pipx install "planoai @ git+https://github.com/TroyMitchell911/plano.git@feat/retry-failover#subdirectory=cli"
```

`pipx` 会自动创建隔离的 venv，并将 `planoai` 命令暴露到全局。

**方案 B：安装到已有的 venv 中**

如果你已经有一个用于其他工具的 Python venv，直接激活它然后安装：

```bash
source /path/to/your/venv/bin/activate
pip install "planoai @ git+https://github.com/TroyMitchell911/plano.git@feat/retry-failover#subdirectory=cli"
```

激活 venv 后 `planoai` 命令即可使用。也可以通过完整路径直接调用：

```bash
/path/to/your/venv/bin/planoai up config.yaml
```

#### 第二步：下载并解压预编译二进制

```bash
# 下载 release tarball
wget https://github.com/TroyMitchell911/plano/releases/download/v0.4.20-retry-failover/plano-v0.4.20-retry-failover-linux-x86_64.tar.gz

# 解压到 ~/.plano — tarball 中包含版本文件，
# 可以防止 CLI 用官方二进制覆盖我们的补丁版
tar xzf plano-v0.4.20-retry-failover-linux-x86_64.tar.gz -C ~/.plano

# 校验 checksum
cd ~/.plano && sha256sum -c checksums.sha256
```

tarball 包含：

```
bin/
  brightstaff          # 补丁版控制面二进制
  envoy                # Envoy v1.37.0 数据面
plugins/
  llm_gateway.wasm     # 补丁版 LLM 网关 WASM filter
  prompt_gateway.wasm  # prompt 网关 WASM filter
patches/
  config_generator.py  # 补丁版 CLI 配置生成器（备用）
checksums.sha256
```

`patches/config_generator.py` 是备用文件——如果你已经从 fork 安装了 CLI，就不需要它。它是为了手动修补 PyPI 安装的情况准备的：

```bash
cp ~/.plano/patches/config_generator.py \
   "$(python3 -c 'import planoai; print(planoai.__path__[0])')/config_generator.py"
```

### Windows / macOS（Docker 模式）

非 Linux 平台上，Plano 在 Docker 容器内运行。你仍然需要安装 Python CLI：

```bash
pip install "planoai @ git+https://github.com/TroyMitchell911/plano.git@feat/retry-failover#subdirectory=cli"
```

然后用 `--docker` 参数启动：

```bash
planoai build --docker
planoai up config.yaml --docker
```

Plano 会构建一个包含 Envoy 和 WASM filter 的 Docker 镜像，然后在容器内运行所有组件。`--docker` 参数在 Linux 上也可以使用，如果你更喜欢容器化部署的话。

> Windows 上需要确保 Docker Desktop 已安装并运行。macOS 上 Docker Desktop 或 OrbStack 都可以。

## 配置

你需要两个文件：`.env` 存放 API key，`config.yaml` 配置路由和供应商。

### .env

在 `config.yaml` 同目录下创建 `.env` 文件：

```bash
# 各渠道的 API key
CLAUDE_API_KEY=sk-your-claude-key
CLAUDE_AWS_API_KEY=sk-your-aws-channel-key
CLAUDE_OFFCIAL_API_KEY=sk-your-official-channel-key
CODEX_API_KEY=sk-your-codex-key
```

每个变量对应一个不同的渠道。如果你的上游是 OpenAI 兼容的 API 聚合器（如 NewAPI / OneAPI），每个 key 通常对应不同的计费组或上游渠道。

### config.yaml（推荐配置）

以下是包含智能路由和多渠道故障转移的完整推荐配置：

```yaml
version: v0.4.0

listeners:
  - type: model
    name: smart_routing
    address: 0.0.0.0
    port: 12000

model_providers:
  - access_key: $CLAUDE_API_KEY
    model: custom/claude-opus-4-6
    base_url: https://your-api-endpoint.com
    provider_interface: openai
    default: true
    retry_policy:
      fallback_models: [custom-aws/claude-opus-4-6, custom-offcial/claude-opus-4-6]
      default_strategy: "different_provider"
      default_max_attempts: 3
      on_status_codes:
        - codes: [401, 429, 503]
          strategy: "different_provider"
          max_attempts: 3

  - access_key: $CLAUDE_API_KEY
    model: custom/claude-sonnet-4-6
    base_url: https://your-api-endpoint.com
    provider_interface: openai

  - access_key: $CLAUDE_AWS_API_KEY
    model: custom-aws/claude-opus-4-6
    base_url: https://your-api-endpoint.com
    provider_interface: openai

  - access_key: $CLAUDE_OFFCIAL_API_KEY
    model: custom-offcial/claude-opus-4-6
    base_url: https://your-api-endpoint.com
    provider_interface: openai

  - access_key: $CODEX_API_KEY
    model: custom/gpt-5.4
    base_url: https://your-api-endpoint.com
    provider_interface: openai

  - access_key: $CODEX_API_KEY
    model: custom/gemini-3-flash-preview
    base_url: https://your-api-endpoint.com
    provider_interface: openai

  - access_key: $CODEX_API_KEY
    model: custom/gemini-3.1-pro-preview
    base_url: https://your-api-endpoint.com
    provider_interface: openai

routing_preferences:
  - name: code generation
    description: >
      generating new code, writing functions, implementing features,
      creating boilerplate, fixing bugs, debugging and patching code
    models:
      - custom/claude-opus-4-6

  - name: code understanding
    description: >
      reading and reviewing existing code, tracing execution flow,
      explaining what code does, code review without modifications
    models:
      - custom/claude-sonnet-4-6

  - name: troubleshooting
    description: >
      analyzing exceptions, diagnosing bugs, investigating errors,
      tracing crash causes, debugging runtime failures, root cause analysis
    models:
      - custom/claude-opus-4-6

  - name: analysis
    description: >
      analyzing data, complex reasoning, evaluating tradeoffs,
      comparing options, strategic planning, architecture decisions
    models:
      - custom/gpt-5.4
      - custom/claude-sonnet-4-6

  - name: documentation
    description: >
      writing documentation, README files, API docs, technical writing,
      user guides, commit messages, comments, changelogs
    models:
      - custom/claude-sonnet-4-6
      - custom/gpt-5.4

  - name: web browsing
    description: >
      browsing websites, fetching web content, searching the internet,
      reading URLs, summarizing web pages
    models:
      - custom/gemini-3-flash-preview

  - name: quick tasks
    description: >
      simple questions, factual lookups, translations, format conversions,
      casual conversation, short answers, trivial tasks
    models:
      - custom/gemini-3-flash-preview
```

### 各字段详解

#### `listeners`

| 字段 | 说明 |
|------|------|
| `type` | `model` — 该 listener 处理 LLM 模型请求 |
| `name` | 可读名称 |
| `address` | 绑定地址。`0.0.0.0` 监听所有网卡 |
| `port` | 应用连接的端口。Plano 在此暴露 OpenAI 兼容 API |

#### `model_providers`

每个条目定义一个可通过 Plano 访问的模型。

| 字段 | 说明 |
|------|------|
| `model` | 模型标识符，格式为 `渠道前缀/模型名`。前缀在转发到上游 API 时会被自动剥离。例如 `custom/claude-opus-4-6` 发送到上游时变为 `claude-opus-4-6` |
| `access_key` | 该渠道的 API key。支持 `$ENV_VAR` 语法从环境变量 / `.env` 文件读取 |
| `base_url` | 上游 API 端点 URL |
| `provider_interface` | API 协议。对任何 OpenAI 兼容端点使用 `openai`（覆盖大多数聚合器） |
| `default` | 设为 `true` 时，当没有路由偏好匹配 prompt 时使用该模型 |

**同模型多渠道：** 你可以将同一个底层模型配置在不同前缀下（如 `custom/claude-opus-4-6`、`custom-aws/claude-opus-4-6`、`custom-offcial/claude-opus-4-6`）。每个有独立的 `access_key`，可以指向不同的计费组或上游渠道。这是跨渠道故障转移的基础。

#### `retry_policy`

附加在特定 model provider 上，控制该渠道失败时的行为。

| 字段 | 说明 |
|------|------|
| `fallback_models` | 有序的备选模型列表，使用完整的 `前缀/模型名` |
| `default_strategy` | `"different_provider"` — 失败时切换到不同渠道 |
| `default_max_attempts` | 最大总尝试次数（包括首次请求） |
| `on_status_codes` | 按 HTTP 状态码的细粒度重试规则 |
| `on_status_codes[].codes` | 触发该规则的 HTTP 状态码列表 |
| `on_status_codes[].strategy` | 这些状态码的重试策略 |
| `on_status_codes[].max_attempts` | 这些特定状态码的最大尝试次数 |

故障转移链路：主模型失败 → 按顺序尝试 `fallback_models` 中的每个模型 → 全部用尽后，尝试尚未尝试过的其他 `model_providers`。

#### `routing_preferences`

这是智能路由的核心。Plano 使用内置的 4B 参数路由模型分析每个 prompt，将其路由到最合适的模型。

| 字段 | 说明 |
|------|------|
| `name` | 类别名称（如 "code generation"、"quick tasks"） |
| `description` | 该类别的自然语言描述。路由模型用它来对传入的 prompt 进行分类 |
| `models` | 该类别使用的模型有序列表。如果列出多个模型，优先使用第一个 |

在上面的推荐配置中：
- **代码生成和故障排查** → Claude Opus（最强推理能力）
- **代码理解和文档编写** → Claude Sonnet（质量和速度的良好平衡）
- **分析** → GPT-5.4 或 Sonnet（强分析能力）
- **网页浏览和简单任务** → Gemini Flash（快速且便宜）

如果 prompt 不匹配任何路由偏好，则使用 `default` 模型。

> **提示：** `routing_preferences` 部分完全是可选的。如果你把它删掉，Plano 依然可以作为纯 failover 代理使用——客户端指定什么模型就走什么模型，`retry_policy` 负责在失败时自动切换到备用渠道。所以即使你不需要智能路由，Plano 也可以当做一个失败自动 fallback 模型的网关来用。

## 启动 Plano

```bash
# 确保 .env 和 config.yaml 在同一目录
cd /path/to/your/config

# 启动（Linux 原生模式）
planoai up config.yaml

# 启动（Docker 模式，适用于 Windows/macOS/Linux）
planoai up config.yaml --docker

# 停止
planoai down
```

启动后，Plano 在配置的端口（默认 `12000`）上监听。将你的工具指向它：

```bash
# 用 curl 测试
curl http://localhost:12000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "custom/claude-opus-4-6",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'
```

任何 OpenAI 兼容客户端都可以使用——只需将 base URL 设为 `http://localhost:12000`，使用配置中的任意模型名。客户端侧甚至不需要真实的 API key；Plano 会在转发到上游之前从 `.env` 注入正确的 key。

## 日志和调试

Plano 日志存储在 `~/.plano/run/logs/`：

```
brightstaff.log    # 控制面日志（路由决策、重试尝试）
envoy.log          # Envoy 数据面日志
access_llm.log     # 所有 LLM 请求的 HTTP 访问日志
```

关键日志模式：

```
# 路由决策
plano-orchestrator determined routes ... response_time_ms=XXX

# 故障转移
Retriable error detected: provider=custom/claude-opus-4-6, status_code=429
Retry initiated: target_model=custom-aws/claude-opus-4-6

# 重试成功
metric.retry_success: model=custom-offcial/claude-opus-4-6, total_attempts=3
```

## 接入 AI Agent

Plano 暴露的是标准 OpenAI 兼容 API，任何支持自定义 base URL 的 agent 都可以直接接入。

对于 [Hermes Agent](https://github.com/hermes-ai/hermes-agent)、[OpenClaw](https://github.com/nicepkg/openclaw)、[Claude Code](https://docs.anthropic.com/en/docs/claude-code) 等工具，只需配置：

- **Base URL：** `http://localhost:12000/v1`
- **API Key：** `.env` 中的任意一个 key 即可（比如 `CLAUDE_API_KEY` 的值）。Plano 会忽略客户端传来的 key，按渠道注入正确的 key，但大多数客户端要求 key 非空才能正常工作。
- **Model：** 配置中的任意模型名（如 `custom/claude-opus-4-6`）。如果配置了智能路由，该模型会被忽略，Plano 会根据任务复杂程度自动路由到相关模型；如果没有配置智能路由，该模型就是被调用的模型，但依然支持 failover。

例如，在一个典型的 agent 配置中：

```yaml
# Hermes Agent config.yaml
providers:
  - name: plano
    kind: openai
    api_key: sk-any...-env
    base_url: http://localhost:12000/v1

models:
  - name: custom/claude-opus-4-6
    provider: plano
```

## 总结

这套配置给你一个 `http://localhost:12000` 端点，它能：

1. 根据 prompt 意图自动选择最合适的模型
2. 主渠道失败时自动切换到备用渠道
3. 遇到限流和服务端错误时指数退避重试
4. 自动剥离渠道前缀，上游 API 收到干净的模型名
5. 兼容任何 OpenAI 客户端，无需改代码

以上推荐配置是我日常使用的配置。根据你自己的供应商调整模型名、API key 和 `base_url` 即可。
