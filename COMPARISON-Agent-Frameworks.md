# OpenJarvis vs. OpenHands vs. GSD vs. LangGraph — Detailed Comparison

A feature comparison of four popular agent tools, written with an eye to **this
machine's setup** (local `qwen-122b`/`qwen-35b` via LiteLLM on an Apple M4 Max).

> **Read this first — they are not the same category.** Comparing their "features"
> head-to-head is a bit apples-to-oranges. They sit at different layers:
>
> | Tool | What it fundamentally *is* |
> |------|----------------------------|
> | **OpenJarvis** | A **personal-assistant runtime/platform** — runs local-first AI agents for *you* (chat, digests, monitors, research), reachable from many channels |
> | **OpenHands** | An **autonomous software-engineering agent** — writes code, runs terminals, opens PRs |
> | **GSD** (Get Shit Done) | A **spec-driven workflow/methodology** layered *on top of* a coding agent (Claude Code, etc.) — it orchestrates the human+AI *process*, it is not a runtime |
> | **LangGraph** | A **low-level library/SDK** for *developers* to build stateful agent graphs in Python/JS |
>
> So the real question is usually "which layer do I need?" not "which is best."

---

## 1. At-a-glance matrix

| Dimension | **OpenJarvis** | **OpenHands** | **GSD** | **LangGraph** |
|-----------|---------------|---------------|---------|---------------|
| **Category** | Personal AI agent platform | Autonomous SWE agent | Dev workflow / meta-prompting system | Agent orchestration library |
| **Primary user** | End user / tinkerer | Developer / team | Developer using Claude Code et al. | Developer building agents |
| **You interact via** | CLI, chat, messaging channels, GUI, HTTP API | Browser "Agent Canvas", SDK, CLI | Slash commands inside Claude Code | Your own code (Python/JS) |
| **Main job** | Be a local personal assistant | Complete coding tasks end-to-end | Keep long projects coherent (beats context rot) | Provide graph/state machinery you program |
| **Local-first / privacy** | ✅ Core design goal (runs on your HW) | ⚠️ Self-hostable, but sandbox + cloud-oriented | ➖ Inherits host agent (e.g. Claude Code → cloud) | ➖ Provider-agnostic; you choose |
| **Runs local models (Ollama/MLX/vLLM)** | ✅ Built-in engines + OpenAI-compat | ✅ Supports Qwen/Llama via LiteLLM | ➖ Whatever the host CLI uses | ✅ Via LangChain provider bindings |
| **Multi-agent orchestration** | ✅ Agents + gateway + operators | ✅ Multi-agent, delegation | ✅ Parallel "wave" subagents | ✅✅ Its core strength (graphs) |
| **Long-term memory** | ✅ Knowledge store (BM25+dense), connectors | ➖ Session/workspace state | ✅ Files on disk (PROJECT/PLAN/phase docs) | ✅ Checkpointers + memory stores |
| **Human-in-the-loop** | Chat-native | Review/approve in UI | ✅ Explicit discuss/verify gates | ✅ First-class (pause/inspect/edit state) |
| **Tool use / code exec** | Tools + MCP + sandboxes | ✅ Full: shell, browser, PRs | via host agent's tools | You wire tools into nodes |
| **Deployment** | Laptop/Mac mini/server, tunnel, channels | Laptop → cloud server (Docker) | N/A (runs in your editor/CLI) | Library → LangGraph Platform/Cloud |
| **How "opinionated"** | Medium (batteries-included) | Medium-high (SWE loop) | High (prescribes the process) | Low (you decide everything) |
| **Coding required to use** | Little (CLI/config) | Little (UI) | None (prompts) | A lot (it's a library) |
| **License / model** | Open source | Open source (+ hosted cloud) | Open source | Open source (+ paid platform) |
| **Rough scale (2026)** | Emerging | ~70k★, SWE-bench ~53% | ~58k★, launched Dec 2025 | v1.0, ~30k★, prod at Uber/LinkedIn/Klarna |

Legend: ✅ strong / built-in · ⚠️ partial or with caveats · ➖ not its focus (possible indirectly)

---

## 2. Tool-by-tool

### OpenJarvis — *your* local personal assistant
- **What it's for:** a private AI that lives on your hardware and does *personal* work —
  chat, morning digests, deep research over your own documents, scheduled monitors,
  and a code assistant, reachable from Telegram/iMessage/Slack/WhatsApp, a desktop GUI,
  or an OpenAI-compatible HTTP API.
- **Standout features:** local-by-default with 13+ inference engines (Ollama, MLX, vLLM,
  llama.cpp, LiteLLM, cloud…), a multi-channel gateway, a personal knowledge store, MCP
  tool support, operators/scheduler for automation, and trace-based learning.
- **Weak spots:** not a specialized coding agent (OpenHands is stronger there); newer and
  smaller ecosystem; you assemble your own workflows rather than following a prescribed method.
- **Fits this machine perfectly:** built to run exactly the local-model stack you already
  have (`qwen-122b` via LiteLLM). See `USING-OpenJarvis.md`.

### OpenHands (formerly OpenDevin) — the autonomous software engineer
- **What it's for:** hand it a GitHub issue or task and it plans, writes code, runs the
  terminal, browses docs, fixes tests, and opens a PR — a full agentic dev loop in a
  sandboxed Docker environment.
- **Standout features:** ~53%+ on SWE-bench Verified with strong models; V1 modular
  **Software Agent SDK** (event-sourced state, opt-in sandboxing, REST/WebSocket server);
  browser "Agent Canvas" UI; VS Code/VNC/browser workspaces; triggerable from Slack/GitHub.
- **Model support:** Claude, GPT, Gemini, and open models (Llama, **Qwen**) via LiteLLM —
  so it *can* run against your local `qwen-122b`, though top scores come from frontier models.
- **Weak spots:** heavier to run (Docker sandbox, ideally a server); focused on software
  engineering, not general personal-assistant tasks; cloud-server deployment is where it shines.

### GSD (Get Shit Done) — the spec-driven *process*
- **What it's for:** keeping **large, multi-session projects coherent**. It's not a runtime;
  it's a set of slash-command workflows for Claude Code (and OpenCode/Gemini CLI/Codex) that
  enforce a **discuss → plan → execute → verify → ship** cycle. *(Installed on this machine —
  see the `/gsd:*` commands and `.planning/`.)*
- **The problem it solves:** *context rot* — quality decay as one long session fills the
  context window. GSD splits work into **phases**, spawns **fresh subagents** with clean
  200k-token contexts, runs independent tasks in parallel "**waves**," and makes `PLAN.md`
  files *the* executable instruction that subagents read directly.
- **Standout features:** interview-driven spec generation, parallel research agents,
  goal-backward verification, persistent `.planning/` docs (PROJECT/REQUIREMENTS/MILESTONES,
  per-phase PLAN/RESEARCH/VERIFICATION). This machine's config: `mode=yolo`, `depth=standard`,
  `parallelization=true`, `model_profile=balanced`.
- **Weak spots:** it's a methodology, so it **inherits the host agent's model/cost/privacy**
  (Claude Code → cloud, unless you point the CLI at a local model); it governs *how you work*,
  it doesn't itself run inference or ship a product runtime.

### LangGraph — the build-it-yourself orchestration library
- **What it's for:** **developers** building custom, stateful, long-running agents. You model
  agent logic as a **graph/state machine**: nodes = actions (LLM calls, tools, sub-agents),
  edges = conditional control flow.
- **Standout features:** durable execution with checkpointing (resume after crashes),
  first-class human-in-the-loop (inspect/modify state mid-run), native token streaming,
  single/multi/hierarchical agent topologies, built-in memory. Stable **v1.0** (late 2025),
  in production at Klarna, LinkedIn, Uber, Replit; DeltaChannel cuts checkpoint storage ~73,000×.
- **Model support:** provider-agnostic through LangChain — including local models, so it can
  target your `qwen` endpoint.
- **Weak spots:** it's infrastructure — **you write the code**; no out-of-the-box assistant,
  coding agent, or UI. Maximum control, maximum effort.

---

## 3. How they relate (they can stack)

These aren't purely competitors — some compose:
- **GSD + OpenHands / Claude Code:** GSD orchestrates the *process*; the coding agent does
  the *work*. (There's literally a `gsd-openhands` project on this machine.)
- **LangGraph inside a product:** you could build OpenJarvis-like or OpenHands-like behavior
  *with* LangGraph as the engine underneath.
- **OpenJarvis as the local brain:** its OpenAI-compatible server can back other tools; and
  OpenHands/LangGraph can point at the *same* local `qwen` endpoint OpenJarvis uses.

The genuine overlaps: **OpenJarvis ↔ OpenHands** both "run agents that do work" (personal vs.
software-engineering focus); **GSD ↔ LangGraph** both address orchestration/state (a prescribed
*method* vs. a programmable *library*).

---

## 4. Which should you use? (given your local-Qwen Mac)

| Your goal | Best pick | Why |
|-----------|-----------|-----|
| A private assistant on your own hardware (chat, digests, research, reachable from your phone) | **OpenJarvis** | Purpose-built local-first; already wired to your `qwen-122b` |
| "Fix this bug / build this feature" autonomously, PRs and all | **OpenHands** | Purpose-built SWE agent; can use your local Qwen via LiteLLM |
| Keep a big, multi-week build coherent without context rot | **GSD** | Prescribes the phase/verify process; already installed here |
| Build your *own* bespoke agent with tight control over state & flow | **LangGraph** | Low-level graph engine; program exactly what you need |

**Rules of thumb**
- Want a *product* to use → OpenJarvis (personal) or OpenHands (coding).
- Want a *process* to follow → GSD.
- Want a *library* to build on → LangGraph.
- **Privacy / offline matters most** → OpenJarvis is the only one designed local-first;
  the others *can* use local models but aren't built around that goal.

---

## Sources
- OpenHands: [openhands.dev](https://www.openhands.dev/) · [GitHub](https://github.com/OpenHands/OpenHands) · [Software Agent SDK (arXiv)](https://arxiv.org/html/2511.03690v1) · [SDK docs](https://docs.openhands.dev/sdk)
- LangGraph: [langchain.com/langgraph](https://www.langchain.com/langgraph) · [GitHub](https://github.com/langchain-ai/langgraph) · [Docs](https://docs.langchain.com/oss/python/langgraph/overview) · [Review 2026](https://toolbrain.net/blog/langgraph-review-2026/)
- GSD: [GitHub (gsd-build/get-shit-done)](https://github.com/gsd-build/get-shit-done) · [Docs](https://gsd-build-get-shit-done.mintlify.app/introduction) · [Augment Code writeup](https://www.augmentcode.com/learn/gsd-58k-stars-claude-code) · [MindStudio guide](https://www.mindstudio.ai/blog/gsd-framework-claude-code-clean-context-phases)
- OpenJarvis: [GitHub (open-jarvis/OpenJarvis)](https://github.com/open-jarvis/OpenJarvis) + local install docs (`SETUP-OpenJarvis.md`, `USING-OpenJarvis.md`, `LLM-STACK-REFERENCE.md`)

*Compiled 2026-07-01. Star counts / benchmark numbers are point-in-time and will drift.*
