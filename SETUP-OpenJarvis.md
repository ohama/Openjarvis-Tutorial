# OpenJarvis Setup — Local qwen-122b on Apple Silicon (M4 Max)

A reproducible guide to installing OpenJarvis and wiring it to a **local Qwen**
model that is already served through a **LiteLLM proxy + MLX** stack. Every step
includes *why*, so you can adapt it if your environment differs.

- **Machine:** Apple M4 Max, 128 GB, macOS 26.x, arm64
- **Goal:** OpenJarvis agents run on local `qwen-122b`, no Ollama, no cloud
- **Result:** verified working (`jarvis ask` returns correct answers via the local model)

---

## 0. Prerequisites — the LLM stack that must already be running

This guide assumes you already run your own inference stack (I did not install it
here — I only connected OpenJarvis to it). For reference, mine is:

| Port | Process | Role |
|------|---------|------|
| `:4000` | `litellm --config agent-stack/litellm/config.yaml` | OpenAI-compatible **proxy / router** (front door) |
| `:8000` | `mlx_lm.server --model .../qwen36-35b` | 35B backend |
| `:8001` | `mlx_lm.server --model .../qwen122b` | 122B backend |
| `:8011` / `:8012` | `role_shim.py` | normalizes system-message ordering for MLX |
| `:8082` | `claude-code-proxy` | Anthropic→chat bridge (unrelated to OpenJarvis) |

The LiteLLM proxy exposes these model **aliases**: `qwen-122b`, `qwen-35b`,
`qwen-local`, `qwen-122b-codex`, `qwen-122b-claude`, `qwen-35b-claude`.

**Verify the stack is up before touching OpenJarvis:**

```bash
# List models the proxy exposes
curl -s http://127.0.0.1:4000/v1/models | python3 -m json.tool

# Confirm the model actually answers
curl -s http://127.0.0.1:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-122b","messages":[{"role":"user","content":"say OK"}],"max_tokens":5}'
```

*Why first:* OpenJarvis is only a client. If `:4000` doesn't answer here, nothing
downstream will work, and you'd waste time debugging the wrong layer.

---

## 1. Understand the two design decisions (read before installing)

OpenJarvis has a pluggable "engine" layer. Two independent choices define the wiring:

### Decision A — talk to the LiteLLM proxy (`:4000`), not raw MLX (`:8001`)

I point OpenJarvis at the **proxy**, not directly at MLX.

*Why:* the proxy is already your front door. Going through it means OpenJarvis
reuses your clean aliases (`qwen-122b`, `qwen-35b`) and your `role_shim` routing.
You get a one-word fallback model and can swap to a role-shim-protected alias by
changing a single string. Going direct to `:8001` would force you to hardcode the
model's **filesystem path** as its name and would bypass the layer you built.

### Decision B — select an "OpenAI-compatible" engine

In OpenJarvis, the engines `vllm`, `sglang`, `llamacpp`, `mlx`, `lmstudio`, … are
**the same code** — each just POSTs to `{host}/v1/chat/completions`. The name only
changes a default port and a label; **it does not launch vLLM or MLX.**

*Why `mlx`:* since your backend really is MLX, use `[engine.mlx]` as the honest
label. (`vllm` would work identically — the name is cosmetic — but `mlx` is
clearer.) The truly generic `openai-compat` engine exists but is **not selectable**
in `config.toml` (only via `jarvis eval --base-url`), so we borrow a registered name.

> Net flow: `OpenJarvis → generic OpenAI HTTP client → LiteLLM :4000 → MLX :8001`

---

## 2. Clone the repository

```bash
cd /Users/ohama/projs/openjarvis-test
git clone --depth 1 https://github.com/open-jarvis/OpenJarvis.git
cd OpenJarvis
```

*Why not the one-line installer* (`curl … install.sh | bash`)*:* it provisions
**Ollama** and pulls a starter model. You already have a better local stack (MLX),
so that step is wasted and would add a competing model server. Installing the pip
package directly skips all of it.

---

## 3. Create a Python 3.13 virtual environment with uv

```bash
uv venv --python 3.13 .venv
```

*Why 3.13 specifically:* OpenJarvis requires `python >=3.10,<3.14`. The system
Python here is **3.14**, which is *too new* and will fail. `uv` downloads and pins a
compatible 3.13 interpreter into the venv without touching system Python.

---

## 4. Install the package (base + server extra)

```bash
VIRTUAL_ENV=.venv uv pip install -e ".[server]"
```

*Why `[server]` and nothing heavier:*
- **Base** deps are pure Python and already include the `openai` client — which is
  all the OpenAI-compatible engine needs. So `jarvis ask` works with the base install.
- **`server`** adds FastAPI/uvicorn so `jarvis serve` / the desktop app can run later.
- I deliberately avoided the **`desktop`** extra: that's the only one pulling the
  **Rust** extension (`openjarvis-rust`), which triggers a compile. Not needed for CLI use.
- Add `tools-search` later (`".[server,tools-search]"`) only if you use web-search tools.

---

## 5. Write the config and install it

OpenJarvis reads `~/.openjarvis/config.toml`. Create this file:

```toml
# OpenJarvis config — wired to local qwen-122b via the LiteLLM proxy (:4000)

[intelligence]
default_model = "qwen-122b"       # LiteLLM alias -> mlx qwen122b (:8001)
fallback_model = "qwen-35b"       # lighter/faster fallback (:8000)
preferred_engine = "mlx"          # OpenAI-compatible client (host points at :4000 below)
provider = "local"
quantization = "none"
temperature = 0.2
max_tokens = 4096
top_p = 0.9

[agent]
default_agent = "orchestrator"    # multi-turn with tool selection
max_turns = 10
context_from_memory = true

[tools]
enabled = ["code_interpreter", "file_read", "file_write", "shell_exec", "web_search", "think", "calculator"]

[tools.storage]
default_backend = "sqlite"
db_path = "~/.openjarvis/memory.db"

[tools.mcp]
enabled = true

[engine]
default = "mlx"                   # any OpenAI-compatible engine name works; mlx is the honest label

[engine.mlx]
host = "http://localhost:4000"    # <-- the LiteLLM proxy, NOT raw mlx :8001
# The proxy currently accepts requests without a client key. If you later enable a
# LiteLLM master key, export it instead of hardcoding:  export MLX_API_KEY="sk-..."
# (env var = <ENGINE_ID>_API_KEY, upper-cased)

[server]
host = "127.0.0.1"
port = 8090                       # moved off :8000 — see why below

[traces]
enabled = true
db_path = "~/.openjarvis/traces.db"
```

Install it:

```bash
mkdir -p ~/.openjarvis
cp config.toml ~/.openjarvis/config.toml
```

*Why these specific values:*
- **`host = :4000`** — Decision A. Sends everything through your proxy.
- **`default_model = "qwen-122b"`** — the proxy alias, not a path. Clean and swappable.
- **`fallback_model = "qwen-35b"`** — a working alias, so degradation is graceful.
- **`[server] port = 8090`** — the OpenJarvis default is `8000`, which **collides with
  your `mlx_lm.server` on `:8000`.** Moving it prevents a startup/port conflict.
- **No API key** — the proxy accepted unauthenticated calls in step 0. The keys inside
  `agent-stack/litellm/config.yaml` are proxy→MLX only; clients don't need them.

---

## 6. Verify the config loads as intended

```bash
.venv/bin/jarvis config show
```

Expect: `Default Engine: mlx`, `Default Model: qwen-122b`,
`Fallback Model: qwen-35b`, `Default Agent: orchestrator`.

*Why:* confirms OpenJarvis parsed the file and resolved the engine/model **before**
you depend on an answer being correct. (`jarvis doctor` runs deeper diagnostics.)

---

## 7. Smoke test

```bash
# 7a. Isolate the model path with the simplest single-turn agent
.venv/bin/jarvis ask --agent simple "Reply with exactly this and nothing else: JARVIS-WIRED-OK"

# 7b. Exercise the configured default (orchestrator) with a reasoning task
.venv/bin/jarvis ask "What is 17 * 23? Reply with just the number."
```

Expected: `JARVIS-WIRED-OK`, then `391`.

*Why two tests:* 7a proves the raw **engine→proxy→model** path in isolation (no tools,
no multi-turn) — if it fails, the problem is the connection, not the agent. 7b then
proves the **default agent** you actually configured works end-to-end.

---

## 8. Daily use

```bash
cd /Users/ohama/projs/openjarvis-test/OpenJarvis
.venv/bin/jarvis ask "..."      # one-shot
.venv/bin/jarvis chat           # interactive session
.venv/bin/jarvis serve          # HTTP API on :8090 (needs the [server] extra)
```

Optional convenience — activate the venv or add it to PATH so you can type bare `jarvis`:

```bash
source .venv/bin/activate       # per-shell
# or symlink: ln -s "$PWD/.venv/bin/jarvis" ~/bin/jarvis
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `jarvis` install fails on Python version | System Python is ≥3.14 | Recreate venv: `uv venv --python 3.13 .venv` (step 3) |
| Connection refused / engine not reachable | LiteLLM `:4000` or MLX backend is down | Re-run the step 0 `curl` checks; restart the stack |
| `System message must be at the beginning` (MLX error) | An agent injected a mid-conversation system message; plain `qwen-122b` skips the role-shim | Set `default_model = "qwen-122b-claude"` (routes through role-shim on `:8011`) |
| Server won't start / port in use | `:8000` collides with `mlx_lm.server` | Ensure `[server] port = 8090` (step 5) |
| `web_search` tool errors | Search extra not installed | `VIRTUAL_ENV=.venv uv pip install -e ".[server,tools-search]"` |

---

## What was installed / touched (for cleanup or audit)

- Repo clone: `/Users/ohama/projs/openjarvis-test/OpenJarvis` (~144 MB)
- Virtualenv: `.venv` inside it (Python 3.13.13, uv-provisioned)
- Config: `~/.openjarvis/config.toml`
- Runtime state (created on use): `~/.openjarvis/memory.db`, `~/.openjarvis/traces.db`
- **Not touched:** your LiteLLM/MLX stack, system Python, Ollama (never installed)

To remove OpenJarvis entirely: delete the clone directory and `~/.openjarvis/`.

---

## Version note

The `main` clone self-reports as a dev build (`0.0.1.dev1+…`) and flags release
`v1.0.3` as available. The dev build tested fine. To use the tagged release instead:

```bash
cd OpenJarvis && git fetch --tags && git checkout v1.0.3 && VIRTUAL_ENV=.venv uv pip install -e ".[server]"
```
