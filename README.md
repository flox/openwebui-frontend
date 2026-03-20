# Open WebUI Frontend

Backend-agnostic [Open WebUI](https://github.com/open-webui/open-webui) v0.8.5 runtime.
Works with any OpenAI-compatible inference server — vLLM, SGLang, Triton, llama.cpp, etc.
Runs on Linux (x86_64) and macOS (Intel & Apple Silicon).

## Prerequisites

- [Flox](https://flox.dev) — a reproducible environment manager
- A running OpenAI-compatible inference backend (vLLM, SGLang, Triton, llama.cpp, Ollama, etc.)
- Supported platforms: `x86_64-linux`, `x86_64-darwin` (Intel Mac), `aarch64-darwin` (Apple Silicon Mac)

## Quick start

Start a backend, then start the frontend pointing at it:

```bash
# Terminal 1 — start a backend
cd path/to/vllm-runtime && flox activate -s    # serves on :8000
# or: cd path/to/sglang-runtime && flox activate -s  # serves on :30000

# Terminal 2 — start the frontend
cd path/to/openwebui-frontend

# For vLLM (port 8000 is the default, just need the API key):
OPENAI_API_KEY=sk-vllm-local-dev flox activate -s

# For SGLang:
BACKEND_PORT=30000 flox activate -s
```

Open WebUI will be available at **http://localhost:8080**.

The service waits for the backend health check to pass before starting,
so it's safe to start both at the same time.

## Configuration

All settings are env vars with sensible defaults. Set them before `flox activate`:

| Variable | Default | Description |
|---|---|---|
| `BACKEND_HOST` | `127.0.0.1` | Inference server host |
| `BACKEND_PORT` | `8000` | Inference server port |
| `BACKEND_HEALTH` | `/health` | Health check endpoint path |
| `OPENAI_API_KEY` | `none` | API key (required if backend enforces auth) |
| `WEBUI_PORT` | `8080` | Open WebUI listen port |
| `DEFAULT_MODEL_PARAMS` | `{"stream_response": false, "max_tokens": 3072}` | Default model parameters (JSON) |
| `CORS_ALLOW_ORIGIN` | `*` | Allowed CORS origins (`;`-separated for multiple) |
| `WEBUI_AUTH` | `false` | Enable Open WebUI authentication |
| `ENABLE_OLLAMA_API` | `false` | Enable Ollama native API (can run alongside OpenAI API) |

## Backend-specific examples

### vLLM

```bash
# Local (port 8000 is the default)
OPENAI_API_KEY=sk-vllm-local-dev flox activate -s

# Remote GPU box
BACKEND_HOST=192.168.0.42 OPENAI_API_KEY=sk-vllm-local-dev flox activate -s
```

vLLM defaults to requiring an API key (`sk-vllm-local-dev` in the standard vllm-flox-runtime).

### SGLang

```bash
# Local
BACKEND_PORT=30000 flox activate -s

# Remote GPU box
BACKEND_HOST=192.168.0.42 BACKEND_PORT=30000 flox activate -s
```

SGLang serves on port 30000 by default and does not require an API key.

### Triton Inference Server

```bash
BACKEND_HOST=triton-server.local BACKEND_PORT=9000 BACKEND_HEALTH=/v1/models flox activate -s
```

Triton uses `/v1/models` as its health endpoint (not `/health`).

### Ollama

```bash
# Ollama only (native API)
BACKEND_PORT=11434 BACKEND_HEALTH=/ ENABLE_OLLAMA_API=true flox activate -s

# Ollama + vLLM (both model sources in one UI)
BACKEND_PORT=8000 OPENAI_API_KEY=sk-vllm-local-dev ENABLE_OLLAMA_API=true flox activate -s
```

With `ENABLE_OLLAMA_API=true`, Open WebUI connects to Ollama's native API
(at `http://localhost:11434` by default) in addition to the OpenAI-compatible
backend. Models from both sources appear in the same interface.

### llama.cpp

```bash
BACKEND_PORT=8080 WEBUI_PORT=8081 flox activate -s
```

llama.cpp defaults to port 8080, which conflicts with Open WebUI's default.
Set `WEBUI_PORT=8081` (or any free port) to avoid the collision.

## How it works

The Flox environment installs the `open-webui-frontend` package (built from
Open WebUI v0.8.5 with patches for configurable model params and streaming).

On activation, the hook:
1. Sets `BACKEND_HOST`/`BACKEND_PORT` from env vars (with defaults)
2. Sources the package's `setup.sh`, which configures Open WebUI env vars
   and creates a Python venv with `uv` (cached, idempotent)
3. Generates a secret key for Open WebUI sessions

The `open-webui` service polls the backend health endpoint, then starts
the Open WebUI uvicorn server.

## First run

The first activation takes longer (~30s) because `uv` creates a Python
venv and installs Open WebUI's dependencies. Subsequent activations
skip this step (cached via requirements hash).

## Troubleshooting

- **Health check never passes** — verify the backend is running and `BACKEND_HEALTH` points to the correct endpoint (e.g. `/v1/models` for Triton instead of the default `/health`)
- **Port conflict** — set `WEBUI_PORT` to an available port (e.g. `WEBUI_PORT=8081`)
- **First run is slow** — expected; the Python venv is being created with `uv` (~30s). Subsequent activations are cached
