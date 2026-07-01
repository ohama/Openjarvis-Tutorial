# Using OpenJarvis — Detailed Guide

How to actually *use* the OpenJarvis install on this machine (wired to local
`qwen-122b` / `qwen-35b` — see `SETUP-OpenJarvis.md` for how it was installed and
`LLM-STACK-REFERENCE.md` for the model endpoint details).

- **CLI binary:** `/Users/ohama/projs/openjarvis-test/OpenJarvis/.venv/bin/jarvis`
- **Config:** `~/.openjarvis/config.toml` (model `qwen-122b`, engine → LiteLLM `:4000`, API server port `8090`)
- Throughout this doc, `jarvis` means that venv binary. To type bare `jarvis`, run
  `source .venv/bin/activate` or symlink it onto your PATH.

> **Includes the answer to "can I use it from my iPhone?" — see section 7.**

---

## 1. The fastest ways to use it

```bash
cd /Users/ohama/projs/openjarvis-test/OpenJarvis

# One-shot question (uses your default agent + qwen-122b)
.venv/bin/jarvis ask "Summarize the theory of relativity in 3 bullets"

# Interactive multi-turn conversation
.venv/bin/jarvis chat

# Guided setup / feature tour for new users
.venv/bin/jarvis quickstart

# Diagnostics — checks config, engine reachability, model, etc.
.venv/bin/jarvis doctor
```

`ask` streams the answer by default. Add `--no-stream` for a single block, or
`--json` to get the raw result object (useful for scripting).

---

## 2. `jarvis ask` — the workhorse

Common options (from `jarvis ask --help`):

| Flag | Purpose |
|------|---------|
| `-m, --model` | Override the model, e.g. `-m qwen-35b` for a faster answer |
| `-a, --agent` | Pick an agent: `simple` (single-turn, no tools), `orchestrator` (multi-turn + tools), `react`, `openhands`. `--agent ''` = talk straight to the model |
| `--tools` | Restrict tools for this call, e.g. `--tools calculator,think` |
| `-t, --temperature` / `--max-tokens` | Sampling overrides |
| `--research` | Answer from your indexed personal knowledge (BM25 + embeddings) |
| `-i, --image FILE` / `-S, --screen` | Send an image / a screenshot to a vision model |
| `--profile` | Print latency / tokens / energy telemetry for the call |
| `--no-context` | Skip injecting memory context |

Examples:
```bash
.venv/bin/jarvis ask -m qwen-35b "quick: capital of Japan?"
.venv/bin/jarvis ask --agent simple --tools calculator "what is 19% of 4,320?"
.venv/bin/jarvis ask -S "what's on my screen right now?"      # needs a vision model
```

---

## 3. Agents, tools, skills

- **Agents** decide *how* a request is handled. `simple` just answers; `orchestrator`
  can call tools across multiple turns; `openhands`/`react` are code/agentic styles.
  Set the default in `config.toml` (`[agent] default_agent`) or override per call.
- **Persistent agents:** `jarvis agents` — create named agents with their own
  persona/config, inspect them, and chat with them.
- **Tools** are capabilities the agent can invoke (`code_interpreter`, `file_read`,
  `file_write`, `shell_exec`, `web_search`, `calculator`, `think`, memory/knowledge
  search…). List them with `jarvis tool list`. Enable per call with `--tools` or
  globally in `[tools] enabled = [...]`.
- **Skills:** `jarvis skill` — reusable saved capabilities. **MCP servers:** add
  external tool servers with `jarvis add` (and `[tools.mcp] enabled = true`).

> Note: some tools need extra installs. `web_search` wants the search extra:
> `VIRTUAL_ENV=.venv uv pip install -e ".[server,tools-search]"`.

---

## 4. Memory, data connections & deep research

OpenJarvis can answer over *your* data:

```bash
# Connect data sources (Gmail, Obsidian, local folders, …)
.venv/bin/jarvis connect            # see available connectors

# Index documents into the knowledge store
.venv/bin/jarvis memory index ~/Documents/papers/

# Ask questions that search your indexed data
.venv/bin/jarvis ask --research "what did the Q2 planning notes conclude?"

# Or the one-shot auto-setup that detects local sources and launches research
.venv/bin/jarvis deep-research-setup
```

Memory/knowledge is stored in `~/.openjarvis/*.db` (sqlite by default, per your config).

---

## 5. Automation — scheduler, operators, digest, workflows

- `jarvis digest` — the "morning digest" (needs a digest config/agent).
- `jarvis scheduler` — schedule recurring tasks (cron-style).
- `jarvis operators` — persistent, scheduled autonomous agents (e.g. monitors).
- `jarvis workflow` — list/run/inspect multi-step workflows.

These turn Jarvis from an on-demand assistant into a background one.

---

## 6. Running it as a server / daemon (foundation for remote & app access)

OpenJarvis can expose an **OpenAI-compatible HTTP API** and run as a background daemon.

```bash
# Foreground server (Ctrl-C to stop)
.venv/bin/jarvis serve                       # binds to config: 127.0.0.1:8090

# Background daemon + lifecycle
.venv/bin/jarvis start                        # start daemon
.venv/bin/jarvis status                       # is it running?
.venv/bin/jarvis restart / stop
```

Once serving, it speaks the standard OpenAI API, so **other software can point at it**
just like it points at LiteLLM:

```bash
curl http://localhost:8090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-122b","messages":[{"role":"user","content":"hi"}]}'
```

> Requires the `server` extra (already installed here). By default it binds to
> loopback only — see the next section to reach it from other devices.

---

## 7. Can I use it from my iPhone?

**Short answer: Yes — but not via a dedicated App Store app.** OpenJarvis ships a
desktop GUI (`.dmg`/`.exe`/`.deb`/…) and a CLI; there is **no native iOS app**. To use
it from an iPhone you connect to the instance running on this Mac, in one of three ways.
The Mac must be **on and the relevant daemon running** for any of them.

### Option A — Messaging channels (easiest, feels native) ✅ recommended

OpenJarvis has a **multi-channel gateway**: you chat with Jarvis inside a messaging app
you already have on your iPhone, and it replies using your local `qwen-122b`.

Supported channels include **Telegram, WhatsApp, Slack, Discord, and iMessage/SMS**
(iMessage/SMS is via the third-party **SendBlue** service).

```bash
.venv/bin/jarvis channel list          # see channel types
.venv/bin/jarvis gateway start         # run the multi-channel gateway daemon
.venv/bin/jarvis gateway status
.venv/bin/jarvis channels imessage-start <CHAT_IDENTIFIER>   # iMessage path
```

- **Telegram** is usually the simplest: create a bot token, put it in the channel
  config, `gateway start`, then message your bot from the Telegram app on your iPhone.
- **iMessage** lets you text Jarvis from the stock Messages app, but needs a SendBlue
  account (paid third party) — heavier to set up.
- **Why this is best for iPhone:** nothing to expose on your network, works from
  anywhere with internet, and the UI is a chat app you already trust.

### Option B — Reach the HTTP API over your network 🌐

Run the server bound to your LAN and hit it from the iPhone. This needs a key
(startup **refuses** a non-loopback bind without one):

```bash
.venv/bin/jarvis auth generate-key                  # create an API key
.venv/bin/jarvis serve --host 0.0.0.0 --port 8090   # bind to the LAN
```

Then from the iPhone (same Wi-Fi):
- Point any **OpenAI-compatible iOS chat app** (or Apple **Shortcuts** with a "Get
  Contents of URL" POST) at `http://<mac-LAN-IP>:8090/v1`, model `qwen-122b`, and send
  the API key as `Authorization: Bearer <key>`.
- Or just open `http://<mac-LAN-IP>:8090/` in Safari if the GUI/web endpoint is served.

Find the Mac's LAN IP with `ipconfig getifaddr en0`. This only works while the iPhone
is on the **same network** as the Mac. For access **away from home**, use a private
mesh VPN like **Tailscale** (install on both devices; use the Mac's Tailscale IP instead
of the LAN IP) — no ports exposed to the public internet.

### Option C — Public tunnel (access from anywhere) 🚇

OpenJarvis has a built-in Cloudflare tunnel:

```bash
.venv/bin/jarvis serve --host 127.0.0.1 --port 8090   # keep it loopback
.venv/bin/jarvis tunnel --port 8090                   # expose via Cloudflare
.venv/bin/jarvis tunnel status
```

This gives a public HTTPS URL you can open from the iPhone anywhere. **Always keep an
API key enabled** when doing this — a tunnel is reachable by anyone with the URL.

### Which should you pick?

| Want | Use |
|------|-----|
| Simplest, chat-app feel, works anywhere | **A — Telegram channel** |
| Text Jarvis from the Messages app | A — iMessage (needs SendBlue) |
| Use an OpenAI-compatible iOS app / Shortcuts, at home | **B — LAN server + key** |
| Same, but also away from home, privately | B + **Tailscale** |
| A public URL from anywhere | **C — Cloudflare tunnel** (+ key) |

### Security reminders for iPhone access
- Never bind to `0.0.0.0` or open a tunnel **without an API key**.
- Prefer Tailscale over port-forwarding your router.
- Your prompts leave the Mac only for the channel/tunnel provider you choose; the model
  itself still runs locally on `qwen-122b`.

---

## 8. Housekeeping

```bash
.venv/bin/jarvis config show      # what config is loaded
.venv/bin/jarvis doctor           # health checks
.venv/bin/jarvis scan             # privacy/security audit of the environment
.venv/bin/jarvis telemetry ...    # latency/token/energy stats of past calls
.venv/bin/jarvis self-update      # upgrade (or use the git/uv method in SETUP doc)
```

**Related docs in this folder:**
- `SETUP-OpenJarvis.md` — how this install was set up (and how to redo it)
- `LLM-STACK-REFERENCE.md` — the LiteLLM → Qwen endpoint reference for any OpenAI-compatible app
