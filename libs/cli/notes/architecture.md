# deepagents-cli Architecture

This document provides an end-to-end breakdown of the `deepagents-cli` package
(`libs/cli/deepagents_cli/`). It traces the call flow from the binary
entry-point down to the LangGraph agent and back, covering every major layer.

---

## 1. Entry point

```text
deepagents / deepagents-cli  →  deepagents_cli:cli_main
```

Both console scripts in `pyproject.toml` resolve to `cli_main` in
`deepagents_cli/__init__.py` via a lazy `__getattr__` redirect that defers the
import of `main.py` until the function is actually called. This keeps
`import deepagents_cli` cheap for code that imports only submodules.

`main.py:cli_main()` is a plain synchronous function. It:

1. Handles the `--version` fast-path without loading any heavy dependency.
2. Calls `check_cli_dependencies()` to verify textual, dotenv, tavily, etc.
3. Calls `parse_args()` to build an `argparse.Namespace`.
4. Branches into one of four execution modes (see §2).

---

## 2. Execution modes

| Flag / condition | Mode | Entry coroutine |
| --- | --- | --- |
| `--acp` | ACP server | `_run_acp_cli_async()` |
| `-n` / `--non-interactive` or piped stdin | Non-interactive | `run_non_interactive()` |
| `update` sub-command / `--update` | Headless update | inline in `cli_main` |
| default | Interactive TUI | `run_textual_cli_async()` |

All async modes are driven by a single top-level `asyncio.run()` call in
`cli_main`.

### 2a. Non-interactive mode (`non_interactive.py`)

- Creates a model, loads MCP tools, wires a LangGraph server, and runs the
  agent to completion.
- Uses `rich.live.Live` for streaming token output directly to stdout/stderr.
- Contains its own inner `while True` / `async for chunk in agent.astream(…)`
  loop identical in structure to the Textual adapter (see §5).

### 2b. Interactive TUI mode

`run_textual_cli_async()` in `main.py`:

1. Resolves the model spec cheaply (no `create_model()` call) so the status
   bar shows the model name on first paint.
2. Builds `server_kwargs` (deferred LangGraph server startup config) and
   `model_kwargs` (deferred `create_model()` config).
3. Calls `run_textual_app()` in `app.py`, which instantiates `DeepAgentsApp`
   and runs it.

---

## 3. Textual application (`app.py`)

`DeepAgentsApp` is a `textual.app.App` subclass. The application lifecycle is:

```text
App.__init__()          store config (agent, thread_id, model, MCP info, …)
App.compose()           build widget tree (VerticalScroll#chat, ChatInput, StatusBar)
App.on_mount()          gc.freeze(), schedule _resolve_git_branch_and_continue
  └─ call_after_refresh → _post_paint_init()   (after first frame is painted)
       ├─ _init_session_state()     create TextualSessionState (thread_id, auto_approve)
       ├─ _start_server_background  spawn LangGraph server subprocess (server_manager.py)
       ├─ _prewarm_threads_cache    pre-populate recent-thread list
       ├─ _prewarm_model_caches     warm-up deferred create_model() call
       ├─ _check_optional_tools_background  warn about missing ripgrep/tavily
       ├─ _check_for_updates        background version check
       └─ _discover_skills          walk skill directories, register slash commands
```

Once the LangGraph server is ready the app emits `DeepAgentsApp.ServerReady`,
which transitions from "Connecting…" to the live session:

```text
on_deep_agents_app_server_ready()
  └─ set _agent, _ui_adapter
  └─ _schedule_initial_submission()   (if -m or --skill was passed)
  └─ drain _pending_messages queue    (messages typed while connecting)
```

### Key state fields

| Field | Type | Purpose |
| --- | --- | --- |
| `_agent` | `RemoteAgent \| Pregel` | LangGraph agent handle |
| `_ui_adapter` | `TextualUIAdapter` | Bridge to streaming execution |
| `_session_state` | `TextualSessionState` | thread_id, auto_approve flag |
| `_agent_running` | `bool` | Guards queue: True while worker runs |
| `_shell_running` | `bool` | Same guard for `!` shell commands |
| `_connecting` | `bool` | True while server is starting up |
| `_pending_messages` | `deque[QueuedMessage]` | FIFO queue during busy states |
| `_processing_pending` | `bool` | Reentrancy guard for queue drain |
| `_agent_worker` | `Worker[None]` | Active Textual background worker |

---

## 4. Widget layer (`widgets/`)

| Widget | File | Role |
| --- | --- | --- |
| `WelcomeBanner` | `welcome.py` | Startup banner with tips, model/MCP summary |
| `ChatInput` | `chat_input.py` | Multi-mode input (normal / shell `!` / command `/`) |
| `StatusBar` | `status.py` | Bottom bar: model, branch, token count, mode |
| `UserMessage` | `messages.py` | Rendered user message bubble |
| `AssistantMessage` | `messages.py` | Streaming markdown response bubble |
| `ToolCallMessage` | `messages.py` | Tool invocation + result display |
| `QueuedUserMessage` | `messages.py` | Ghost bubble for queued messages |
| `ErrorMessage` | `messages.py` | Red error display |
| `ApprovalMenu` | `approval.py` | HITL approve/reject/edit pop-up |
| `AskUserMenu` | `ask_user.py` | Structured question input pop-up |
| `LoadingWidget` | `loading.py` | Spinner overlay during startup |
| `ModelSelector` | `model_selector.py` | `/model` modal for live model switching |
| `ThreadSelector` | `thread_selector.py` | `/threads` modal for thread browsing |

---

## 5. Agent layer (`agent.py`, `server_manager.py`, `remote_client.py`)

### Server architecture

In the default interactive mode the CLI spawns a child process running
`langgraph dev` (via `server_manager.py`). The Textual app connects to it
through the LangGraph SDK `RemoteAgent` client (`remote_client.py`), which
wraps `langgraph_sdk.RemoteGraph`.

In ACP mode and certain test configurations a local `Pregel` graph is used
directly without a subprocess.

### Agent creation (`agent.py:create_cli_agent`)

The agent graph is composed using `deepagents.create_deep_agent()` with a
stack of middleware:

| Middleware | Purpose |
| --- | --- |
| `LocalContextMiddleware` | Injects cwd, git status, OS context into every prompt |
| `MemoryMiddleware` | Manages long-term memory retrieval/storage |
| `SkillsMiddleware` | Wires runnable skill sub-graphs |
| `ConfigurableModelMiddleware` | Allows live model switching mid-session |
| `ShellAllowListMiddleware` | Validates shell commands against the allow-list |

Tools registered per run:

- Built-in: `read_file`, `write_file`, `edit_file`, `run_shell`, `grep`,
  `fetch_url`, `web_search`, and others in `tools.py`.
- MCP tools: loaded via `langchain-mcp-adapters` from `~/.deepagents/mcp.json`
  or a project-level config.

---

## 6. Slash command registry (`command_registry.py`)

Every slash command is a `SlashCommand` dataclass in the `COMMANDS` tuple. The
`BypassTier` enum controls whether a command can bypass the message queue while
the agent is running:

| Tier | Behaviour |
| --- | --- |
| `ALWAYS` | Execute immediately even mid-thread-switch (`/quit`, `/q`) |
| `CONNECTING` | Bypass only during server startup, not during agent/shell |
| `IMMEDIATE_UI` | Open a modal immediately; real work deferred via `_defer_action` |
| `SIDE_EFFECT_FREE` | Execute immediately; defer chat output until idle |
| `QUEUED` | Wait in queue when the app is busy |

---

## 7. MCP integration (`mcp_tools.py`, `mcp_trust.py`)

MCP servers are resolved at startup by `resolve_and_load_mcp_tools()`:

1. Walk global (`~/.deepagents/mcp.json`), project (`.deepagents/mcp.json`),
   and explicit (`--mcp-config`) config files.
2. Check `mcp_trust.py` for stdio server trust decisions (prompt user if
   unknown).
3. Initialize `langchain-mcp-adapters` session managers, collect tool schemas,
   return `(tools, session_manager, server_info)`.

The session manager is kept alive for the duration of the process.

---

## 8. Skills (`skills/`, `built_in_skills/`)

Skills are portable agent workflows stored as directories containing a
`SKILL.md` (schema + prompt) and optional Python/JS scripts. `_discover_skills`
walks `~/.deepagents/skills/`, project `.deepagents/skills/`, and the bundled
`built_in_skills/` directory.

Invoking `/skill-name` calls `_invoke_skill()` in `app.py`, which loads the
skill content, mounts a `SkillMessage` widget, and calls `_send_to_agent()`
with injected skill metadata.

---

## 9. Session persistence (`sessions.py`, `config.py`)

Thread IDs are UUID strings generated by `generate_thread_id()`. The LangGraph
SQLite checkpointer (via `langgraph-checkpoint-sqlite`) persists the graph
state. `thread_exists()` queries the checkpointer to decide whether to show the
post-session resume hint.

---

## 10. Hook system (`hooks.py`)

`dispatch_hook(event, payload)` fires registered async callbacks for lifecycle
events such as `session.start`, `user.prompt`, `task.complete`,
`context.offload`. Hooks are lightweight and optional; unregistered events are
no-ops.

---

## Module map (quick reference)

```text
deepagents_cli/
├── __init__.py          Lazy entry-point redirect (cli_main)
├── main.py              Arg parsing, mode dispatch, run_textual_cli_async
├── app.py               DeepAgentsApp (Textual), dispatch loop, HITL wiring
├── textual_adapter.py   execute_task_textual, streaming loop, UI callbacks
├── non_interactive.py   run_non_interactive, headless streaming loop
├── agent.py             create_cli_agent, middleware stack
├── server_manager.py    LangGraph server subprocess lifecycle
├── remote_client.py     RemoteAgent (wraps langgraph_sdk.RemoteGraph)
├── command_registry.py  SlashCommand, BypassTier, COMMANDS
├── tools.py             Built-in tool definitions
├── mcp_tools.py         MCP server loading, resolve_and_load_mcp_tools
├── mcp_trust.py         Stdio trust prompting & persistence
├── config.py            Settings, create_model, build_stream_config, console
├── model_config.py      ModelSpec, PROVIDER_API_KEY_ENV, recent model
├── sessions.py          generate_thread_id, thread_exists, list_threads
├── skills/              Skill discovery & loading utilities
├── built_in_skills/     Bundled skills (remember, skill-creator)
├── hooks.py             Lifecycle event dispatch
├── formatting.py        Rich/Textual formatting helpers
├── output.py            JSON output mode, OutputFormat
└── widgets/             All Textual widget definitions
    ├── chat_input.py    ChatInput, multi-mode input box
    ├── messages.py      Message widgets (User, Assistant, Tool, Error, …)
    ├── status.py        StatusBar
    ├── welcome.py       WelcomeBanner, startup tips
    ├── approval.py      ApprovalMenu (HITL)
    ├── ask_user.py      AskUserMenu (structured questions)
    └── …
```
