+++
date = '2026-04-27T16:59:39+08:00'
draft = false
title = 'Plano: Smart LLM Router with Failover and Intelligent Routing'
tags = ['LLM', 'AI']
summary = 'Deploy Plano as a local LLM proxy with automatic failover across providers, per-status-code retry policies, and intent-based smart routing — so your requests always land on the right model through the right channel.'
+++

## What is Plano?

[Plano](https://github.com/katanemo/plano) is an open-source, local LLM gateway that sits between your application and upstream LLM providers. It speaks the OpenAI-compatible API, so any tool that can talk to OpenAI can talk to Plano — no code changes needed.

Out of the box, Plano already supports multi-model routing and basic provider management. On top of that, I've added patches for:

- **Automatic retry** on rate-limit (429) and server errors (5xx), with exponential backoff and `Retry-After` header support
- **Cross-provider failover** — when one provider channel fails, Plano automatically tries the next one in your fallback list
- **Multi-provider same-model** — configure the same underlying model (e.g. `claude-opus-4-6`) under multiple provider prefixes with independent credentials
- **Intent-based smart routing** — Plano analyzes the prompt and routes to the best model for the task (code generation → Opus, quick questions → Flash, etc.)

The patched release is available at [TroyMitchell911/plano](https://github.com/TroyMitchell911/plano/releases/tag/v0.4.20-retry-failover).

## Installation

### Linux (native mode)

#### Step 1: Install the CLI

The CLI (`planoai`) is a Python package that generates Envoy configs and manages the Plano lifecycle.

**If `pip install` works on your system:**

```bash
pip install "planoai @ git+https://github.com/TroyMitchell911/plano.git@feat/retry-failover#subdirectory=cli"
```

> You must install from the fork, not from PyPI. The upstream `planoai==0.4.20` on PyPI does not support multi-provider same-model configurations and will error out with `Duplicate model_id`.

**If your system blocks `pip install` (externally-managed-environment):**

Many modern Linux distros (Arch, Fedora 38+, Ubuntu 23.04+, Debian 12+) use PEP 668 to lock down the system Python. You'll see:

```
error: externally-managed-environment
```

Two ways around this:

**Option A: pipx (recommended)**

```bash
pipx install "planoai @ git+https://github.com/TroyMitchell911/plano.git@feat/retry-failover#subdirectory=cli"
```

`pipx` creates an isolated venv automatically and exposes the `planoai` command globally.

**Option B: Install into an existing venv**

If you already have a Python venv for other tools, just activate it and install there:

```bash
source /path/to/your/venv/bin/activate
pip install "planoai @ git+https://github.com/TroyMitchell911/plano.git@feat/retry-failover#subdirectory=cli"
```

The `planoai` command will be available whenever that venv is active. You can also call it directly via the full path:

```bash
/path/to/your/venv/bin/planoai up config.yaml
```

#### Step 2: Download and extract the prebuilt binaries

```bash
# Download the release tarball
wget https://github.com/TroyMitchell911/plano/releases/download/v0.4.20-retry-failover/plano-v0.4.20-retry-failover-linux-x86_64.tar.gz

# Extract to ~/.plano — the tarball includes version files that prevent
# the CLI from re-downloading official binaries over our patched ones
tar xzf plano-v0.4.20-retry-failover-linux-x86_64.tar.gz -C ~/.plano

# Verify checksums
cd ~/.plano && sha256sum -c checksums.sha256
```

The tarball contains:

```
bin/
  brightstaff          # patched control plane binary
  envoy                # Envoy v1.37.0 data plane
plugins/
  llm_gateway.wasm     # patched LLM gateway WASM filter
  prompt_gateway.wasm  # prompt gateway WASM filter
patches/
  config_generator.py  # patched CLI config generator (backup)
checksums.sha256
```

The `patches/config_generator.py` is a backup — if you installed the CLI from the fork, you don't need it. It's there in case you need to manually patch an existing PyPI installation:

```bash
cp ~/.plano/patches/config_generator.py \
   "$(python3 -c 'import planoai; print(planoai.__path__[0])')/config_generator.py"
```

### Windows / macOS (Docker mode)

On non-Linux platforms, Plano runs inside Docker. You still need the Python CLI installed:

```bash
pip install "planoai @ git+https://github.com/TroyMitchell911/plano.git@feat/retry-failover#subdirectory=cli"
```

Then start Plano with the `--docker` flag:

```bash
planoai build --docker
planoai up config.yaml --docker
```

Plano will build a Docker image containing Envoy and the WASM filters, then run everything inside a container. The `--docker` flag works on Linux too if you prefer containerized deployment.

> On Windows, make sure Docker Desktop is installed and running. On macOS, Docker Desktop or OrbStack both work.

## Configuration

You need two files: `.env` for API keys and `config.yaml` for routing and provider setup.

### .env

Create a `.env` file alongside your `config.yaml`:

```bash
# Primary provider keys
CLAUDE_API_KEY=sk-your-claude-key
CLAUDE_AWS_API_KEY=sk-your-aws-channel-key
CLAUDE_OFFCIAL_API_KEY=sk-your-official-channel-key
CODEX_API_KEY=sk-your-codex-key
```

Each variable corresponds to a different provider channel. If your upstream is an OpenAI-compatible API aggregator (like NewAPI / OneAPI), each key typically maps to a different billing group or upstream channel.

### config.yaml (recommended configuration)

Here's the full recommended configuration with smart routing and multi-provider failover:

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

### Field-by-field breakdown

#### `listeners`

| Field | Description |
|-------|-------------|
| `type` | `model` — this listener handles LLM model requests |
| `name` | A human-readable name for this listener |
| `address` | Bind address. `0.0.0.0` listens on all interfaces |
| `port` | The port your applications connect to. Plano exposes an OpenAI-compatible API here |

#### `model_providers`

Each entry defines a model accessible through Plano.

| Field | Description |
|-------|-------------|
| `model` | Model identifier in `provider-prefix/model-name` format. The prefix before `/` is stripped when forwarding to the upstream API. For example, `custom/claude-opus-4-6` sends `claude-opus-4-6` to the upstream |
| `access_key` | API key for this provider channel. Supports `$ENV_VAR` syntax to read from environment / `.env` file |
| `base_url` | The upstream API endpoint URL |
| `provider_interface` | API protocol. Use `openai` for any OpenAI-compatible endpoint (covers most aggregators) |
| `default` | If `true`, this model is used when no routing preference matches the prompt |

**Multi-provider same-model:** You can configure the same underlying model under different prefixes (e.g. `custom/claude-opus-4-6`, `custom-aws/claude-opus-4-6`, `custom-offcial/claude-opus-4-6`). Each has its own `access_key`, so they can point to different billing groups or upstream channels. This is the foundation for cross-provider failover.

#### `retry_policy`

Attached to a specific model provider, controls what happens when that provider fails.

| Field | Description |
|-------|-------------|
| `fallback_models` | Ordered list of alternative models to try. Uses full `prefix/model` names |
| `default_strategy` | `"different_provider"` — switch to a different provider on failure |
| `default_max_attempts` | Maximum total attempts (including the initial request) |
| `on_status_codes` | Fine-grained retry rules per HTTP status code |
| `on_status_codes[].codes` | List of HTTP status codes that trigger this rule |
| `on_status_codes[].strategy` | Retry strategy for these codes |
| `on_status_codes[].max_attempts` | Max attempts for these specific codes |

The failover chain works like this: primary model fails → try each model in `fallback_models` in order → if all exhausted, try remaining `model_providers` that haven't been attempted yet.

#### `routing_preferences`

This is where the smart routing magic happens. Plano uses a built-in 4B parameter routing model to analyze each prompt and route it to the best model.

| Field | Description |
|-------|-------------|
| `name` | Category name (e.g. "code generation", "quick tasks") |
| `description` | Natural language description of what this category covers. The routing model uses this to classify incoming prompts |
| `models` | Ordered list of models to use for this category. If multiple models are listed, the first one is preferred |

In the recommended config above:
- **Code generation and troubleshooting** → Claude Opus (strongest reasoning)
- **Code understanding and documentation** → Claude Sonnet (good balance of quality and speed)
- **Analysis** → GPT-5.4 or Sonnet (strong analytical capabilities)
- **Web browsing and quick tasks** → Gemini Flash (fast and cheap)

If a prompt doesn't match any routing preference, it goes to the `default` model.

> **Tip:** The `routing_preferences` section is entirely optional. If you remove it, Plano still works as a pure failover proxy — all requests go to the `default` model, and the `retry_policy` handles automatic fallback across providers on failure. So if you don't need smart routing, Plano is still useful as a self-healing LLM gateway that automatically switches providers when one goes down.

## Starting Plano

```bash
# Make sure .env is in the same directory as config.yaml
cd /path/to/your/config

# Start (Linux native)
planoai up config.yaml

# Start (Docker mode, for Windows/macOS/Linux)
planoai up config.yaml --docker

# Stop
planoai down
```

Once running, Plano listens on the configured port (default `12000`). Point your tools at it:

```bash
# Test with curl
curl http://localhost:12000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "custom/claude-opus-4-6",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'
```

Any OpenAI-compatible client works — just set the base URL to `http://localhost:12000` and use any model name from your config. You don't even need a real API key on the client side; Plano injects the correct key from your `.env` before forwarding to the upstream.

## Logs and debugging

Plano logs are stored in `~/.plano/run/logs/`:

```
brightstaff.log    # control plane logs (routing decisions, retry attempts)
envoy.log          # Envoy data plane logs
access_llm.log     # HTTP access log for all LLM requests
```

Key log patterns to look for:

```
# Routing decision
plano-orchestrator determined routes ... response_time_ms=XXX

# Failover in action
Retriable error detected: provider=custom/claude-opus-4-6, status_code=429
Retry initiated: target_model=custom-aws/claude-opus-4-6

# Retry success
metric.retry_success: model=custom-offcial/claude-opus-4-6, total_attempts=3
```

## Connecting AI agents

Since Plano exposes a standard OpenAI-compatible API, any agent that supports a custom base URL can connect directly.

For [Hermes Agent](https://github.com/hermes-ai/hermes-agent), [OpenClaw](https://github.com/nicepkg/openclaw), [Claude Code](https://docs.anthropic.com/en/docs/claude-code), or similar tools, just configure:

- **Base URL:** `http://localhost:12000/v1`
- **API Key:** any key from your `.env` (e.g. the value of `CLAUDE_API_KEY`). Plano ignores the client-side key and injects the correct one per provider, but most clients require a non-empty key to function.
- **Model:** any model name from your config (e.g. `custom/claude-opus-4-6`). If smart routing is configured, this model is ignored — Plano automatically routes to the appropriate model based on task complexity. If smart routing is not configured, this is the model that gets called, but failover is still supported.

For example, in a typical agent config:

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

## Summary

This setup gives you a single `http://localhost:12000` endpoint that:

1. Automatically picks the best model for each prompt based on intent
2. Falls back to alternative provider channels if the primary one fails
3. Retries on rate-limits and server errors with exponential backoff
4. Strips provider prefixes so the upstream API receives clean model names
5. Works with any OpenAI-compatible client — no code changes needed

The recommended config above is what I use daily. Adjust the model names, API keys, and `base_url` to match your own provider setup.
