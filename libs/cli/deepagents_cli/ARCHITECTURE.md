# deepagents CLI — Core Architecture

This document explains the core construction of `deepagents_cli` from source
code: how the binary starts up, parses commands, wires the agent graph,
integrates tools, manages state/sessions, streams output, and compares to
Claude Code's `QueryEngine`.

---

## 1. Entrypoint Wiring

```
pyproject.toml → [project.scripts]
  deepagents       = "deepagents_cli:cli_main"
  deepagents-cli   = "deepagents_cli:cli_main"
```

### `__init__.py` — lazy gateway

`deepagents_cli/__init__.py` exports `cli_main` via `__getattr__` so that
the heavy startup machinery in `main.py` (argparse, signal handling, etc.)
is never loaded when other submodules — `config`, `widgets`, `theme` — are
imported in isolation.

```python
# __init__.py
def __getattr__(name):
    if name == "cli_main":
        from deepagents_cli.main import cli_main
        return cli_main
    raise AttributeError(...)
```

### `main.py` — `cli_main()`

`cli_main()` is the actual entry point. It follows a strict fast-path / lazy
import discipline to keep startup latency low:

1. **Fast path for `-v`/`--version`**: prints version without loading any
   heavy module.
2. `check_cli_dependencies()` — validates that `textual`, `requests`, etc.
   are installed, exiting early with a friendly message if not.
3. `parse_args()` — dispatches to the argparse tree.
4. Imports `console` and `settings` *only after* arg-parsing (so `--help`
   never pays the settings bootstrap cost).
5. Branches based on `args.command` or the presence of
   `args.non_interactive_message`:
   - Subcommands (`agents`, `threads`, `skills`, `update`) → headless handlers.
   - `--acp` → ACP server mode (`_run_acp_cli_async`).
   - `-n TEXT` → `run_non_interactive` (single-shot, no TUI).
   - Everything else → `run_textual_cli_async` (interactive TUI).

---

## 2. Command Parsing

### Argparse tree

`parse_args()` builds an `argparse.ArgumentParser` with subcommands:

| Subcommand | Purpose |
|------------|---------|
| `help` | Rich help screen via `ui.show_help()` |
| `agents list / reset` | Manage per-agent `AGENTS.md` prompts |
| `skills` | Install / list / remove agent skills |
| `threads list / delete` | Browse and clean up conversation threads |
| `update` | Self-update via pip/uv |

Top-level flags (when no subcommand is given) control the interactive or
non-interactive session:

| Flag | Effect |
|------|--------|
| `-a / --agent NAME` | Select agent (default: `"agent"`) |
| `-M / --model MODEL` | Override LLM (auto-detects provider) |
| `-m / --message TEXT` | Auto-submit prompt on TUI start |
| `-n / --non-interactive TEXT` | Single-shot headless run then exit |
| `-r / --resume [ID]` | Resume last or specific thread |
| `-y / --auto-approve` | Skip all HITL confirmation prompts |
| `--sandbox TYPE` | Run tools in a remote sandbox |
| `--mcp-config PATH` | Extra MCP server config |
| `--no-mcp` | Disable MCP tool loading |

`-h` / `--help` on every subcommand triggers a dedicated Rich help screen
(via a custom `argparse.Action` closure), not argparse's default plain-text
output.

### Slash commands (in-session)

Runtime slash commands are declared once in
`command_registry.py → COMMANDS: tuple[SlashCommand, ...]`. Each entry
carries:

- `name`: canonical name, e.g. `/model`.
- `bypass_tier`: controls whether the command waits in the message queue or
  fires immediately. Tiers:

  | Tier | Behavior |
  |------|----------|
  | `ALWAYS` | Fires regardless of busy state (e.g. `/quit`) |
  | `CONNECTING` | Bypasses only during initial server startup |
  | `IMMEDIATE_UI` | Opens a modal immediately; real work deferred |
  | `SIDE_EFFECT_FREE` | Effect fires now; chat output waits for idle |
  | `QUEUED` | Must wait while the agent is running |

Bypass-tier frozensets (`ALWAYS_IMMEDIATE`, `IMMEDIATE_UI`, etc.) are
derived automatically from `COMMANDS` — no other file hard-codes command
metadata.

---

## 3. Agent / Runtime Orchestration

### 3a. `create_cli_agent()` — agent graph construction

`agent.py → create_cli_agent()` is the single factory used by every mode
(interactive TUI, non-interactive, ACP). It:

1. **Assembles a middleware stack** (in order):
   - `ConfigurableModelMiddleware` — allows mid-session `/model` switching.
   - `TokenStateMiddleware` — persists `_context_tokens` in the checkpoint.
   - `AskUserMiddleware` (optional) — adds `ask_user` interrupt tool.
   - `MemoryMiddleware` — reads/writes `AGENTS.md` for persistent memory.
   - `SkillsMiddleware` — discovers and injects skills from multiple
     directories (built-in → user → project).
   - `LocalContextMiddleware` — detects git state, directory layout, MCP
     metadata and injects them into the system prompt.
   - `ShellAllowListMiddleware` (optional) — validates shell commands inline
     against an allow-list without HITL interrupts.
   - `SummarizationToolMiddleware` — adds `compact_conversation` tool to
     offload older messages.

2. **Creates a backend** (local vs. sandbox):
   - Local: `LocalShellBackend` (filesystem + shell) or `FilesystemBackend`
     (no shell).
   - Remote: sandbox backend (`ModalSandbox`, `DaytonaSandbox`, etc.).
   - Wraps in `CompositeBackend` with routing for large results and
     conversation history.

3. **Configures HITL interrupt_on map** via `_add_interrupt_on()`:
   Gated tools (require user approval unless `--auto-approve`):
   `execute`, `write_file`, `edit_file`, `web_search`, `fetch_url`, `task`,
   `launch_async_subagent`, `update_async_subagent`, `cancel_async_subagent`,
   `compact_conversation`.

4. **Calls `create_deep_agent()`** (SDK entry point) with the model,
   system prompt, tools, backend, middleware, `interrupt_on`, checkpointer,
   and subagent definitions.

5. Returns `(agent_graph: Pregel, composite_backend: CompositeBackend)`.

### 3b. Server-based interactive mode

The interactive TUI does **not** run the agent graph in-process. It:

1. Spawns a `langgraph dev` subprocess
   (`server_manager.start_server_and_get_agent`).
2. Connects via `langgraph-sdk` (`RemoteAgent`) over HTTP.
3. Sends user messages as streaming API requests to the local server.

The `auto_approve` flag is **not** passed to the server; the server always
configures full HITL interrupts. The client-side `session_state.auto_approve`
flag in `textual_adapter.py` controls whether the TUI silently confirms
interrupts without showing the approval menu.

### 3c. Non-interactive mode

`non_interactive.py → run_non_interactive()` also uses `server_session`
(langgraph dev subprocess). The key difference is that with a restrictive
`--shell-allow-list`, shell validation is done inline by
`ShellAllowListMiddleware` instead of interrupts, keeping the LangSmith
trace as a single continuous run.

---

## 4. Tool Integration

### Built-in tools

Defined in `tools.py`:

| Tool | Description |
|------|-------------|
| `web_search` | Tavily-powered web search (requires `TAVILY_API_KEY`) |
| `fetch_url` | Fetches and Markdown-converts a URL |

The SDK's `create_deep_agent` / backend layer supplies the filesystem and
shell tools (`read_file`, `write_file`, `edit_file`, `list_dir`,
`execute`, etc.).

### MCP tools

`mcp_tools.py → resolve_and_load_mcp_tools()` discovers, loads, and adapts
MCP servers to LangChain tools via `langchain-mcp-adapters`. Sources:

- Auto-discovered configs (Claude Desktop format) in user and project
  directories.
- Explicit `--mcp-config PATH`.

Project-level stdio servers require interactive approval the first time
(trust is stored in `mcp_trust.py`).

### HITL gate (`_add_interrupt_on`)

Every tool with side effects is gated at the LangGraph level:
`interrupt_on: dict[str, InterruptOnConfig]`. When the agent emits a tool
call for a gated tool, LangGraph pauses the graph and emits an
`__interrupt__` event. The TUI renders an `ApprovalMenu`; the user's
decision is sent back as a `Command(resume=...)`.

---

## 5. State / Session Handling

### Thread identity

Sessions are identified by a UUID7 thread ID (generated by
`sessions.generate_thread_id()`). The ID is passed to LangGraph as
`config["configurable"]["thread_id"]`.

### Checkpointing

LangGraph's `AsyncSqliteSaver` backs the interactive TUI session
(`~/.deepagents/sessions.db`). `InMemorySaver` is used for ACP mode and
some short-lived test scenarios.

### `TextualSessionState`

`app.py → TextualSessionState` holds the client-side session:

- `thread_id` — current thread (reset with a UUID7 on `/clear`).
- `auto_approve` — toggled with `Shift+Tab`.

### Token state

`token_state.py → TokenStateMiddleware` persists `_context_tokens` in the
graph state checkpoint so the status bar can display accurate token counts
when resuming a thread.

### Thread browser (`sessions.py`)

Provides async functions to list (`list_threads_command`), delete
(`delete_thread_command`), and inspect threads stored in the SQLite
checkpoint database. Results are cached in `_recent_threads_cache` to avoid
redundant DB hits.

---

## 6. Streaming / Output Pipeline

### Interactive TUI (`textual_adapter.py`)

`execute_task_textual()` drives the streaming loop:

```
agent.astream(input, stream_mode=["messages", "updates"], subgraphs=True)
  ↓
for chunk (namespace, stream_mode, data) in stream:
    if "updates":
        → detect __interrupt__ → queue HITL/ask_user decisions
        → detect todo updates
    elif "messages":
        → skip subagent namespaces (ns_key != ())
        → AIMessageChunk  → accumulate text per namespace → AssistantMessage widget
        → AIMessageChunk with tool_calls → ToolCallMessage widget (pending)
        → ToolMessage → update tool widget (success/error)
        → HumanMessage → flush pending text
        → summarization chunks → show "Offloading..." spinner
```

After the stream ends (or an interrupt fires), the loop either:
- Returns `SessionStats` on completion.
- Re-enters with `Command(resume=decision)` after the user responds to an
  interrupt.

### Non-interactive mode (`non_interactive.py`)

`run_non_interactive()` drives a similar streaming loop but renders to
stdout/stderr via Rich `Live` display instead of Textual widgets:

- Streaming text tokens written directly to `sys.stdout`.
- Tool calls shown as Rich spinners.
- `--quiet` redirects all Rich console output to stderr; only the final
  response text goes to stdout.
- `--no-stream` buffers the full response and prints it at once.

### Subagent filtering

Both TUI and non-interactive streams filter out subagent output by checking
the chunk namespace: `ns_key == ()` means the main agent; non-empty
namespaces are subagents. Subagent text is hidden from the chat view but
contributes to the main agent's eventual response.

### Summarization feedback

When `MemoryMiddleware` offloads old messages, chunks are tagged with
`metadata["lc_source"] == "summarization"`. Both adapters detect this tag
and show a `"Offloading..."` spinner rather than displaying the
summarization model's intermediate output to the user.

---

## 7. Key Module / Control Flow Map

```
cli_main() [main.py]
  │
  ├─ parse_args() → argparse namespace
  ├─ apply_stdin_pipe() → stdin merge into args
  │
  ├─[subcommand]─► headless command handler (list/delete/reset/update)
  │
  ├─[--acp]──────► _run_acp_cli_async()
  │                  ├─ create_model()
  │                  ├─ resolve_and_load_mcp_tools()
  │                  ├─ create_cli_agent()  [agent.py]
  │                  └─ run_acp_agent() [deepagents-acp]
  │
  ├─[-n TEXT]────► run_non_interactive()  [non_interactive.py]
  │                  ├─ server_session()  [server_manager.py]
  │                  │   └─ start_server_and_get_agent()
  │                  │       ├─ ServerConfig.to_env()
  │                  │       ├─ scaffold workspace (langgraph.json)
  │                  │       └─ langgraph dev subprocess → RemoteAgent
  │                  └─ streaming loop
  │                      ├─ agent.astream(input, ...)
  │                      ├─ render tokens / tool widgets (Rich Live)
  │                      └─ HITL auto-approve loop
  │
  └─[interactive]► run_textual_cli_async()  [main.py]
                     ├─ resolve model spec (fast, no langchain import)
                     └─ run_textual_app()  [app.py]
                         ├─ DeepAgentsApp.__init__()
                         │   ├─ _start_server_background() [worker]
                         │   │   ├─ start_server_and_get_agent()
                         │   │   └─ post ServerReady message
                         │   └─ _load_model_background() [worker]
                         │       └─ create_model() → apply_to_settings()
                         │
                         └─ on_chat_input() → _handle_command() / _process_user_input()
                             ├─ slash command routing (command_registry.py)
                             └─ execute_task_textual()  [textual_adapter.py]
                                 ├─ build_stream_config(thread_id, assistant_id)
                                 ├─ agent.astream(input, stream_mode=["messages","updates"])
                                 ├─ mount AssistantMessage / ToolCallMessage widgets
                                 ├─ HITL interrupt → ApprovalMenu → Command(resume=)
                                 └─ return SessionStats
```

---

## 8. Comparison to Claude Code QueryEngine

| Dimension | deepagents CLI | Claude Code QueryEngine |
|-----------|---------------|------------------------|
| **Scheduling model** | Graph-driven, LangGraph `Pregel` node loop. Agent can spawn subagents (`task` tool) in parallel sub-graphs. | Single async generator main loop (`submitMessage → for await query()`). No graph; control stays in one coroutine. |
| **State scope** | Graph-level checkpoint (`AsyncSqliteSaver`) holding all messages plus custom channels (`_context_tokens`, `todos`, `memory`). Survives across sessions. | Session-scoped mutable arrays (`mutableMessages`, `permissionDenials`, `readFileState`). Flushed to `SessionStorage`. |
| **Tool gate** | `interrupt_on` map in LangGraph. When triggered, graph pauses; client resumes with `Command(resume=decision)`. | `canUseTool → wrappedCanUseTool`. Gate is inline in the event loop; denials recorded in `permissionDenials`. |
| **HITL pattern** | Graph interrupt + `Command(resume=...)` from the client. Splits into multiple LangSmith trace segments unless `interrupt_shell_only` is used. | Inline continuation — loop never pauses; decisions are applied within the same streaming turn. |
| **Streaming event model** | LangGraph `astream(stream_mode=["messages","updates"], subgraphs=True)` yields `(namespace, mode, data)` 3-tuples. Namespaces separate main agent from subagents. | SDK emits typed events: `message_start/delta/stop`, `api_retry`, `stream_event`. Each event has a `subtype` for fine-grained handling. |
| **Cost / budget control** | No per-session USD budget. Token count tracked via `_context_tokens` in checkpoint; visible in status bar. | Hard budget cap (`maxBudgetUsd`, `totalCost`). Loop exits with `error_max_budget_usd` result subtype. |
| **Compact / offload** | `compact_conversation` tool triggers `SummarizationToolMiddleware`; full history in SQLite, LLM-generated summary replaces older messages in context. | `compact_boundary` / `snip` replay pattern. Older messages trimmed inline; compressed state replayed into new request. |
| **Retry / resilience** | LangGraph handles node-level retries; API retries delegated to langchain-core. | Explicit `categorizeRetryableAPIError` + configurable `maxRetries` in the main loop. |
| **Session persistence** | First-class: SQLite checkpoint + `thread_id`; threads listed/resumed via `deepagents threads list` or `-r`. | `recordTranscript` / `flushSessionStorage` for JSON transcript files; explicit `compact_boundary` snip replay. |
| **Multi-agent** | Native: `task` tool spawns subagents in child sub-graphs; `async_subagents` dispatch to remote LangGraph deployments. | Single-agent. No native sub-graph delegation (custom tools can call external APIs). |
| **One-liner summary** | **Workflow / graph orchestration layer** — "who runs next and how do agents collaborate?" | **Conversation runtime execution layer** — "how does one turn run reliably, recoverably, within budget?" |
