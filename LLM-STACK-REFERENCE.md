# Local LLM Stack Reference — LiteLLM → Qwen 122B / 35B (MLX)

**Purpose:** everything another app needs to talk to the local Qwen models on this
machine. If a piece of software can point at an **OpenAI-compatible** endpoint, this
document alone is enough to configure it — no need to inspect the stack yourself.

- **Host:** Apple M4 Max, 128 GB, macOS, arm64
- **Serving:** Apple **MLX** (`mlx_lm.server`) behind a **LiteLLM** proxy
- **One thing to remember:** point your app at **`http://localhost:4000/v1`** and pick a model name from the table below.

---

## TL;DR — the only two values most apps need

```
OpenAI-compatible Base URL : http://localhost:4000/v1
API key                    : (none required — send any non-empty string if the field is mandatory)
Model name                 : qwen-122b   (heavy)   |   qwen-35b   (fast)
```

> **OpenAI-compatible:** the base URL above implements the standard OpenAI REST API
> (`/v1/models`, `/v1/chat/completions`, streaming via SSE). Any software that supports
> a custom OpenAI base URL / "OpenAI-compatible provider" (OpenJarvis, Open WebUI,
> LibreChat, Continue, Cursor, LangChain, the `openai` SDK, etc.) works by setting that
> URL and a model name — nothing MLX-specific is exposed to the client.

---

## Architecture

```
                 ┌────────────────────────────────────────────────────────────┐
   Your app ───► │  LiteLLM proxy      :4000   (OpenAI-compatible front door)   │
 (OpenAI API)    └───────┬───────────────────────────┬────────────────────────┘
                         │                            │
          plain chat     │                            │  role-shim path (system-msg safe)
                         ▼                            ▼
             ┌──────────────────────┐     ┌──────────────────────────────┐
             │ mlx_lm.server :8001  │     │ role_shim :8011 ──► :8001     │  (122B)
             │  qwen122b  (122B)    │     │ role_shim :8012 ──► :8000     │  (35B)
             ├──────────────────────┤     └──────────────────────────────┘
             │ mlx_lm.server :8000  │
             │  qwen36-35b (35B)    │
             └──────────────────────┘

  (claude-code-proxy :8082 sits in front for Anthropic-API clients — see note below)
```

Everything runs on `127.0.0.1` (loopback only). The LiteLLM proxy is the intended
entry point; the `:8000`/`:8001` MLX servers and the shims are internal.

---

## Model catalog (LiteLLM aliases on `:4000`)

| Model name | Size | Routes to | Path through | Use it for |
|------------|------|-----------|--------------|------------|
| `qwen-122b` | 122B | MLX `:8001` | direct | Best quality / agents / reasoning |
| `qwen-35b` | 35B | MLX `:8000` | direct | Fast / lightweight / high-throughput |
| `qwen-local` | 35B | MLX `:8000` | direct | Legacy alias for `qwen-35b` |
| `qwen-122b-claude` | 122B | MLX `:8001` | via role-shim `:8011` | 122B when the client sends system/developer messages out of order |
| `qwen-35b-claude` | 35B | MLX `:8000` | via role-shim `:8012` | 35B, same role-safe path |
| `qwen-122b-codex` | 122B | MLX `:8001` | Responses→chat bridge + role-shim | OpenAI **Responses API** / Codex-style clients |

**Which to pick:**
- Normal OpenAI-compatible app → **`qwen-122b`** or **`qwen-35b`**.
- If you hit the MLX error *"System message must be at the beginning"* (some agent
  frameworks inject system messages mid-conversation) → switch to the **`*-claude`**
  variant, which normalizes message roles.
- If your client speaks the **Responses API** (not chat/completions) → **`qwen-122b-codex`**.

---

## Serving parameters (fixed at launch)

| Backend | Model dir | Port | Max tokens | Thinking |
|---------|-----------|------|-----------|----------|
| 122B | `/Users/ohama/llm-system/models/qwen122b` | `:8001` | 16384 | disabled (`enable_thinking:false`) |
| 35B | `/Users/ohama/llm-system/models/qwen36-35b` | `:8000` | 8192 | disabled (`enable_thinking:false`) |

- `max-tokens` is the server-side ceiling for **output** length; requesting more is clamped.
- Thinking/reasoning traces are disabled server-side, so responses are the final answer only.

---

## Verify the stack is up (run before configuring any app)

```bash
# 1. Is the proxy answering, and what models does it expose?
curl -s http://localhost:4000/v1/models | python3 -m json.tool

# 2. Does the 122B model actually generate?
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-122b","messages":[{"role":"user","content":"say OK"}],"max_tokens":5}'

# 3. Same check for the 35B
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-35b","messages":[{"role":"user","content":"say OK"}],"max_tokens":5}'
```

If (1) lists the aliases and (2)/(3) return a completion, any OpenAI-compatible app
will work. If not, the problem is the stack, not your app.

---

## How to configure common OpenAI-compatible software

Set the base URL and model; leave the key blank or use any placeholder.

**Generic (env vars used by the `openai` SDK, LangChain, many tools):**
```bash
export OPENAI_BASE_URL="http://localhost:4000/v1"
export OPENAI_API_KEY="local"        # any non-empty value; the proxy ignores it
# then use model = "qwen-122b"
```

**Python `openai` SDK:**
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:4000/v1", api_key="local")
resp = client.chat.completions.create(
    model="qwen-122b",
    messages=[{"role": "user", "content": "Hello"}],
)
```

**TOML/YAML-config apps (e.g. OpenJarvis's OpenAI-compatible engine):**
```toml
# OpenAI-compatible engine → point host at the LiteLLM proxy, pick a model alias
[engine]
default = "mlx"                     # any "OpenAI-compatible" engine label works
[engine.mlx]
host = "http://localhost:4000"      # NOT the raw MLX port; the proxy is the front door
# model = "qwen-122b"
# API key only needed if you later enable a LiteLLM master key (see below)
```

**Rule of thumb:** wherever the app asks for an "OpenAI API base URL", "custom endpoint",
or "OpenAI-compatible provider", enter `http://localhost:4000/v1` and choose a model
name from the catalog.

---

## Notes & gotchas

- **No auth today.** The proxy currently accepts unauthenticated requests. The API keys
  inside `agent-stack/litellm/config.yaml` are used by LiteLLM to reach the MLX backends
  — clients don't need them. If you later enable a LiteLLM **master key**, every app must
  then send `Authorization: Bearer <key>`.
- **`/v1` suffix.** Include `/v1` when the app wants a full base URL. Some apps append it
  themselves — if you see `/v1/v1` in errors, drop the suffix.
- **Port `:8000` is taken** by the 35B MLX server. If you run another app with a server of
  its own, bind it to a different port.
- **Loopback only.** Nothing is exposed to the LAN. To serve other devices you'd change the
  proxy bind address and add a key — out of scope here.
- **Anthropic-API clients** (e.g. Claude Code) can use `claude-code-proxy` on `:8082`, which
  converts Anthropic → chat and forwards into this same stack. Standard apps should use the
  OpenAI-compatible `:4000` path above.

---

## Component map (for maintenance/debugging only — apps don't need this)

| Component | Command | Port | Config / logs |
|-----------|---------|------|---------------|
| LiteLLM proxy | `agent-stack/venv/bin/litellm --config agent-stack/litellm/config.yaml --port 4000` | `:4000` | `agent-stack/litellm/config.yaml` |
| MLX 122B | `mlx_lm.server --model .../qwen122b --port 8001 --max-tokens 16384` | `:8001` | `llm-system/services/logs/122b.*` |
| MLX 35B | `mlx_lm.server --model .../qwen36-35b --port 8000 --max-tokens 8192` | `:8000` | `llm-system/services/logs/36-35b.*` |
| Role-shim (122B) | `role_shim.py --listen 8011 --upstream http://localhost:8001` | `:8011` | `llm-system/services/logs/role-shim.*` |
| Role-shim (35B) | `role_shim.py --listen 8012 --upstream http://localhost:8000` | `:8012` | `llm-system/services/logs/role-shim-35b.*` |
| Claude-code proxy | `claude-code-proxy -s` | `:8082` | `agent-stack/claude-code-proxy/` |

**Role-shim purpose:** `mlx_lm.server` requires any system message to be the first
message and rejects the OpenAI Responses `developer` role. The shim rewrites messages
(first `system`/`developer` → `system` kept at front; later ones → `user`) and forwards
everything else byte-for-byte, including streaming. That's why the `*-claude` / `-codex`
aliases exist.
