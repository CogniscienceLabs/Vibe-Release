# Vibe AI Assistant
> **v0.8.103** — A Jarvis-like, multi-agent AI assistant that thinks, plans, codes, researches, and controls your desktop autonomously.

---

## Table of Contents

1. [What is Vibe?](#what-is-vibe)
2. [System Architecture](#system-architecture)
3. [Quick Start Guide](#quick-start-guide)
4. [Features](#features)
5. [Configuration Reference](#configuration-reference)
6. [Plugin System (PDK)](#plugin-system-pdk)
7. [CLI Reference](#cli-reference)
8. [Changelog](#changelog)

---

## What is Vibe?

Vibe is a **local-first, Jarvis-style AI assistant** built around a hierarchical multi-agent architecture. It runs as a native desktop application (Electron GUI + Python backend) and combines the reasoning power of large language models with actual computer control — it can browse the web, write and execute code, manage files, click UI elements, and chain these capabilities together to autonomously complete complex, multi-step goals.

**What makes Vibe different from a chatbot:**
- **LLM-agnostic** — swap providers per-agent: Gemini, OpenAI, Anthropic, xAI/Grok, OpenRouter, or local Ollama models. No lock-in.
- **OS-agnostic desktop control** — Computer Use Agent runs natively on Windows, macOS, and Linux. VM mode (VirtualBox/VNC) available for sandboxed execution on any host OS.
- **It actually does things** — controls your real desktop via vision + mouse/keyboard, not just generating instructions for you to follow
- **Full code execution loop** — writes, runs, and self-reviews code; CoderAgent has direct filesystem access and a post-run CodeReview Agent pass
- **Programmatic Tool Calling (PTC)** — for batch data tasks, the CoderAgent writes and executes a script instead of ping-ponging through LLM calls. 80-95% token savings on data-heavy operations. Auto-routes to host (safe) or VM sandbox (network/untrusted)
- **Multi-step agentic planning** — MA breaks goals into subtasks with dependency resolution, parallel dispatch, and automatic self-correction on failure
- **Agent collaboration** — Research→CUA browser fallback ("Vibe Loop"), Code→CodeReview pipeline, inter-agent clarification requests
- **4-tier memory** — grep-based fast context, event log, HNSW vector store, and per-subagent in-session operational log for zero-latency within-task continuity
- **Safety-first** — path allowlist/blacklist, command safety validation, safe mode, VM-only mode, ENV-only key storage, log redaction
- **Fully extensible** — Plugin Development Kit (PDK) for custom agents, capabilities, and in-app config UI

---

## System Architecture

Vibe runs a **Python backend** (ZMQ message bus + asyncio task pipeline) connected to an **Electron/React GUI** (real-time event feed). Five specialized agents are coordinated by a central Managerial Agent (MA).

### Agent Hierarchy

```
User
 └── Managerial Agent (MA)     ← Brain. Plans, routes, synthesizes, clarifies.
       ├── Research Agent       ← Web search, URL analysis, info synthesis
       ├── Coder Agent          ← Code generation, debugging, full filesystem I/O
       ├── Computer Use Agent   ← Desktop vision + mouse/keyboard control
       ├── Image Agent          ← Image generation & analysis (Gemini Imagen)
       └── Code Review Agent    ← Post-coding quality, security, best-practices
```

### Managerial Agent (MA)
The only agent the user talks to directly. It classifies requests (direct response vs. complex plan), creates structured subtask graphs with dependency resolution, runs subtasks in parallel where possible, recovers from failures autonomously, and synthesizes a final response. If needed, it pauses mid-execution to ask the user a clarifying question (with clickable option cards) without ending the session.

### Computer Use Agent (CUA)
Vibe's "hands." Uses Gemini vision to identify UI elements, then acts via:
- `pyautogui` for click/type/scroll/drag (with **DPI-aware coordinate scaling**)
- `pywinauto` / `atomacos` / `pyatspi` for OS-native focus and keyboard injection
- Win32 API (`SetForegroundWindow`, `AttachThreadInput`) for reliable window control
- **Two-tier verification** — Tier 1: API foreground state (~50ms). Tier 2: vision before/after (~1-2s). Auto-escalates on failure.
- **Full goal-completion check** — after success, `verify_goal_complete()` asks the VLM "is the ENTIRE goal actually done?"
- **VM Mode** — run inside VirtualBox via VNC (raw RFB client, zero deps) for sandboxed operation

### Task Pipeline

```
User Message → MA classifies
  ├─ Simple → Direct response
  ├─ Ambiguous → Clarifying question (mid-turn, non-blocking)
  └─ Complex → Execution plan (subtasks + dependencies)
                  ↓
           TaskManager dispatches
          (parallel where no deps)
                  ↓
       Specialized agents execute
       → emit live feed events
       → return results
                  ↓
         MA synthesizes → Final response
```

### Storage Layout
```
~/.vibe_ai/
  config.json               ← Settings & ENV:VAR_NAME key refs
  .env                      ← Actual secrets (never committed)
  script_trust_store.json   ← Saved script fingerprint approvals (session/workspace/global)
  session_histories/
    <session_id>/
      history.json          ← Conversation turns
      notebook.md           ← MA's notes & task checklist
      session.md            ← User-editable session instructions
  memory/
    events.jsonl            ← Temporal event log (Tier 2)
    persistent/             ← personality.md & project notes
  pdks/pdks.json            ← Installed plugin registry
```
On first run, Vibe creates `~/.vibe_ai/` with defaults. Open **Settings → Keys** and point the Gemini key field to your env var.

### Example Prompts
| Request | Agent(s) used |
|---|---|
| *"Summarize recent news on quantum computing"* | Research |
| *"Write a Python script to batch-rename files"* | Coder + Code Review |
| *"Open Chrome and navigate to github.com"* | CUA |
| *"Open YouTube and play a lo-fi music video"* | CUA (multi-step) |
| *"Generate an image of a cyberpunk city at night"* | Image |

### Adding More Providers
In **Settings → Keys**, add keys for additional providers:

| Provider | Env Variable | Notable Models |
|---|---|---|
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| xAI | `XAI_API_KEY` | 
| Ollama/LMstudio | *(no key)* | 
| Qwen | Oauth or API key |

Local Ollama: run `ollama pull llama3.3`, then set any agent to provider `Ollama` in Settings → Models.

---

## Features

### Multi-Agent Orchestration
- **Parallel execution** — independent subtasks run concurrently; dependency-ordered tasks chain sequentially
- **Autonomous recovery** — failed subtasks trigger MA replanning without user intervention
- **Research→CUA fallback ("Vibe Loop")** — web scraping failure escalates to CUA browser automation automatically
- **Mid-turn clarification** — agent pauses, shows question + option cards, resumes with user's answer

### Computer Use Agent
- Full desktop control: click, type, scroll, drag, key combos, multi-monitor awareness
- DPI-aware coordinate scaling between `mss` physical capture and `pyautogui` logical coordinates
- Accessibility API typing (`acc_type_to_window`) — no OS focus required
- Smart click focus heuristics — visible in-target clicks can execute directly without wasteful pre-focus hops; explicit focus-first is reserved for backgrounded or occluded windows
- Active control overlay — the `[vibe]` cursor badge is now explicitly enabled around interactive host-control actions for clearer user awareness
- Path blacklist — user-defined dirs permanently blocked from agent access
- Path allowlist — override system-protected paths for approved directories
- VM Mode (Host / VM / Auto) with per-session override and hot-switch
- Active-surface capture routing — screenshots, verification, waits, and initial CUA context stay bound to the host desktop or sandbox surface that the task is actually running on
- Sandbox lifecycle visibility — Docker/VirtualBox startup, resume, ready, and VNC-connect phases emit normalized status events that surface in the feed instead of disappearing into backend logs

### Memory & Context
- **4-tier hybrid memory**: grep (Tier 1) + temporal JSONL event log (Tier 2) + HNSW vector store (Tier 3) + subagent operational log (Tier 4) — Tiers 1-3 queried in parallel and fusion-ranked by MA; Tier 4 is a zero-latency in-memory rolling event log injected into every subagent LLM call for within-session continuity (what it just did, what failed, what files it modified)
- **Session-scoped RAG**: per-session + cross-session `__overall__` memories queried simultaneously
- **Context caching**: Gemini `cachedContent` (1hr TTL), Anthropic `cache_control` breakpoints, OpenAI automatic prefix caching — all fail-safe
- **Session-aware specialist call surfaces**: HistoryManager, Computer Use text/vision flows, and ImageAgent generate/edit/analyze paths now route through centralized helper methods so caching, session attribution, reasoning config, and token accounting stay consistent instead of fragmenting across ad-hoc micro-calls
- **Race-safe MA task context propagation**: Managerial Agent subtask guidance is now carried through message metadata and bound task-locally inside specialist `handle_message_async()` entrypoints via `ContextVar`-backed agent state, preventing same-agent parallel subtasks from overwriting each other's prompt context
- **Auto-checkpointing**: summarize + prune older turns when nearing context window, with persisted checkpoint artifacts that now track manual vs automatic runs, snapshot vs prune outcomes, tokens saved, turns pruned, per-agent compression totals, and a structured `checkpoint_state` payload (`objective`, `current_state`, `work_completed`, `key_decisions_constraints`, `open_threads_todos`, `next_planned_actions`) for higher-fidelity compaction continuity
- **Per-model context limits**: `MODEL_CONTEXT_WINDOWS` covers all providers; truncation is model-accurate

### LLM Support
- **Providers**: Gemini, OpenAI, Anthropic, xAI/Grok, OpenRouter, Ollama
- **Per-agent config**: each agent has its own provider, model, temperature, reasoning effort
- **Model hot-swap**: switch mid-session without losing history; context window recalculates
- **Runtime presets**: save full agent configs as named presets, swap instantly

### GUI Highlights
- Real-time event feed with collapsible "Task Completed" / "Planning & Execution" cards
- Full Markdown rendering (GFM: code blocks, tables, links)
- Session Dashboard (grid of all sessions with preview/message count)
- Session Config Panel (non-blocking side panel: preset selector, per-agent temps, session instructions)
- Agent-triggered workspace expansion — backend session scope can now append roots programmatically, and GUI feed events can add directories directly into the active workspace tree
- Script approval cards with trust-scope selection (`session`, `workspace`, `global`) and saved-trust badges for previously approved matching scripts
- Agent HUD for live CUA status (vision/action state)
- Standalone sandbox lifecycle cards for VM/Docker bring-up and connection state
- Token usage display per session, including context-window-aware compression totals and per-agent checkpoint breakdowns in the context panel
- Inline Monaco code editor opened by agents for file review

### Safety
- **Path blacklist** — permanent block on user-defined sensitive directories
- **Path allowlist** — per-directory override for normally-protected system paths
- **Windows protected path guard** — `System32`, `SysWOW64`, `%WINDIR%` always blocked
- **Command safety validator** — regex-based BLOCKED / REQUIRE_CONFIRMATION / SAFE for shell commands
- **Script fingerprint trust store** — approved scripts can be remembered by fingerprint at `session`, `workspace`, or `global` scope so repeated safe runs do not require another prompt
- **Safe mode** — CUA read-only; VM-only mode blocks all host operations
- **ENV-only key storage** — secrets in `~/.vibe_ai/.env`, only references in `config.json`
- **Log redaction** — API key patterns scrubbed from all log output before disk

---

## Configuration Reference

All settings managed via the **Settings** modal — no manual JSON editing needed.

### Key Config Options

| Key | Default | Description |
|---|---|---|
| `safe_mode` | `true` | Read-only CUA; blocks destructive file ops |
| `script_trust_store_enabled` | `true` | Enables saved script fingerprint approvals and trust-scope reuse |
| `debug_mode` | `false` | Show raw agent events in timeline |
| `cua_control_mode` | `"host"` | `host`, `vm`, or `auto` |
| `vm_only_mode` | `false` | Block all host CUA ops |
| `unsafe_mode_path_allowlist` | `[]` | File/folder paths exempt from system protection |
| `command_allowlist` | `[]` | Executables/commands allowed in regulated mode |
| `unsafe_mode_path_blacklist` | `[]` | Dirs permanently blocked from agent access |
| `auto_checkpoint_enabled` | `true` | Auto-prune context near window limit |
| `api_retry_max_retries` | `5` | Retries for transient API errors |
| `code_review_mode` | `"auto"` | `auto` (complexity-gated), `always`, `off` |
| `code_review_complexity` | `"medium"` | Auto-mode threshold: `low`, `medium`, `high`, `critical` |
| `vm_auto_rules` | `""` | User guidelines for auto-mode risk classification |

### Agent Temperatures (recommended defaults)

| Agent | Temp |
|---|---|
| Managerial | 0.5 |
| Coder | 0.75 |
| Research | 0.7 |
| Computer Use | 0.2 |
| Image | 0.9 |
| Code Review | 0.3 |

### Reasoning Effort
Per-agent: `none` / `low` / `medium` / `high`. Maps to Gemini `thinking_budget`, Anthropic `budget_tokens`, OpenAI `reasoning_effort`. Configurable per-agent from Settings → Models or per-session from the Config panel.

### Session Instructions
Each session has a `session.md` at `~/.vibe_ai/session_histories/<session_id>/session.md`. Edit via the **Config** side panel or directly on disk. Agents read it at the start of every request. The MA can also self-edit `session.md` autonomously via the `update_session_instructions` notebook action (`mode: append` or `mode: replace`) — useful for persisting project facts, user preferences, or workflow notes discovered mid-session.

### Script Approval Trust Scopes
When Vibe asks to run a script from the CoderAgent/PTC flow, you can still do a normal one-click approve or deny. If the script requires approval and the trust store is enabled, the approval card can also persist that decision for future matching scripts:

- **Session** — trust the script fingerprint only for the current session
- **Workspace** — trust the script fingerprint for the current workspace path
- **Global** — trust the script fingerprint everywhere on this machine

Trust matches are based on a stable script fingerprint derived from the script's execution-relevant metadata. The trust store does not bypass blocked scripts; it only suppresses repeated approval prompts for matching scripts that would otherwise require confirmation.

### Local Runtimes
```json
{
  "ollama_base_url": "http://127.0.0.1:11434",
  "openai_compatible_base_url": "http://127.0.0.1:8000"
}
```

---

## Plugin System (PDK)

Plugins add new agent capabilities. They expose a simple `activate(ctx)` contract:

```python
# my_plugin/main.py
def activate(ctx):
    ctx.register_capability(
        name="my_tool",
        description="Does something useful",
        handler=lambda params: {"result": params["input"]},
        parameters_schema={"input": {"type": "string"}}
    )
```

**`pdk.json` manifest** declares `id`, `name`, `version`, `entrypoint`, `capabilities`, `permissions`, and optional `config_schema` (string/secret/number/boolean fields rendered dynamically in the Plugins UI).

**Install**: Plugins modal → Install from Folder → pick directory → toggle Enabled. No restart needed. The MA's planning prompt updates immediately.
