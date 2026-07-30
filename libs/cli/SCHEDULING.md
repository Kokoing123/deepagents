# deepagents CLI — Scheduling Loop

This document traces the full scheduling loop of `deepagents_cli` from user
input through agent invocation, streaming, tool/HITL handling, and output
rendering. Code references use exact file paths and function names from the
`libs/cli/deepagents_cli/` source tree.

---

## 1. Entry Points

Two primary execution modes are available once `cli_main()` in `main.py`
dispatches based on flags:

| Mode | Entry function | Location |
| ---- | -------------- | -------- |
| **Interactive TUI** | `run_textual_cli_async()` → `DeepAgentsApp` | `main.py`, `app.py` |
| **Non-interactive** | `run_non_interactive()` | `non_interactive.py` |

Both modes share the same agent graph (`Pregel` or `RemoteAgent`) and the
same `astream(stream_mode=["messages","updates"], subgraphs=True)` call.

---

## 2. Interactive TUI Scheduling Loop

### 2a. Input capture

```text
ChatInput widget  →  ChatInput.Submitted event
  │
  ▼
DeepAgentsApp.on_chat_input_submitted()  [app.py:2228]
  │
  ├── [/quit or /q]  →  self.exit()                  (always-immediate path)
  │
  ├── [_thread_switching]  →  notify + return        (thread-switch guard)
  │
  ├── [_agent_running or _connecting]
  │     ├── [bypass-eligible command]  →  _process_message() immediately
  │     └── otherwise  →  _pending_messages.append(QueuedMessage)
  │                        mount QueuedUserMessage widget, return
  │
  └── _process_message(value, mode)                  [app.py:2148]
```

### 2b. Input routing (`_process_message`)

```text
_process_message(value, mode)  [app.py:2148]
  │
  ├── mode == "shell"    →  _handle_shell_command()  (! prefix, separate worker)
  ├── mode == "command"  →  _handle_command()        [app.py:2613]
  │                          (handles /clear, /model, /offload, /threads, etc.)
  └── mode == "normal"   →  _handle_user_message()   [app.py:3231]
```

### 2c. Agent submission (`_handle_user_message` → `_send_to_agent`)

```text
_handle_user_message(message)  [app.py:3231]
  │
  ├── mount UserMessage widget
  └── _send_to_agent(message)                        [app.py:3241]
        │
        ├── anchor VerticalScroll to bottom
        ├── self._agent_running = True
        ├── disable ChatInput cursor
        └── self.run_worker(                         (@work / Textual Worker)
              _run_agent_task(message)               [app.py:3284]
            )
```

`run_worker()` spawns a Textual background worker so the main event loop
stays responsive (key events, spinner animation, scroll) while the agent
streams.

### 2d. Worker: `_run_agent_task`

```text
_run_agent_task(message)  [app.py:3284]
  │
  ├── turn_stats = SessionStats()           (pre-created for cancellation safety)
  ├── _inflight_turn_stats = turn_stats
  │
  ├── await execute_task_textual(           [textual_adapter.py:355]
  │     user_input=message,
  │     agent=self._agent,                  (Pregel or RemoteAgent)
  │     session_state=self._session_state,
  │     adapter=self._ui_adapter,
  │     turn_stats=turn_stats,
  │     context=CLIContext(model_override),
  │     ...
  │   )
  │
  ├── [on exception]  →  finalize_pending_tools_with_error()
  │                       mount ErrorMessage widget
  └── [finally]
        ├── merge turn_stats → _session_stats
        └── _cleanup_agent_task()
              ├── _agent_running = False
              ├── set_spinner(None)
              ├── restore ChatInput cursor
              └── _process_next_from_queue()  (drain pending messages)
```

### 2e. Core stream loop: `execute_task_textual`

```text
execute_task_textual(...)  [textual_adapter.py:355]
  │
  ├── parse_file_mentions(user_input)       (@ filename injection)
  ├── create_multimodal_content(...)        (images/videos)
  ├── config = build_stream_config(thread_id, assistant_id)
  ├── set_spinner("Thinking")
  │
  └── while True:                           ← OUTER HITL LOOP
        │
        ├── interrupt_occurred = False
        ├── pending_interrupts = {}
        ├── pending_ask_user = {}
        │
        └── async for chunk in agent.astream(   ← INNER STREAM LOOP
              stream_input,
              stream_mode=["messages", "updates"],
              subgraphs=True,
              config=config,
              durability="exit",
            ):
              │
              ├── namespace, stream_mode, data = chunk
              ├── is_main_agent = (namespace == ())
              │
              ├── stream_mode == "updates":
              │     └── "__interrupt__" in data
              │           ├── type=="ask_user"  →  pending_ask_user[id]
              │           └── HITL request     →  pending_interrupts[id]
              │                                    interrupt_occurred = True
              │
              └── stream_mode == "messages":
                    ├── [not is_main_agent]  →  skip (subagent output filtered)
                    │
                    ├── [summarization chunk]  →  spinner("Offloading"), skip
                    │
                    ├── HumanMessage     →  flush pending text for namespace
                    │
                    ├── ToolMessage      →  pop from _current_tool_messages
                    │                       set_success / set_error on widget
                    │                       show DiffMessage if file modified
                    │
                    └── AIMessageChunk   →  accumulate text per namespace
                          ├── text block     →  AssistantMessage.append_content()
                          └── tool_call block →  mount ToolCallMessage widget
                                                  track in _current_tool_messages
              │
              └── [after inner loop]
                    ├── flush remaining pending text for all namespaces
                    │
                    └── [interrupt_occurred]
                          ├── ask_user interrupts  →  adapter._request_ask_user()
                          │                            await AskUserMenu result
                          ├── HITL interrupts      →  adapter._request_hitl()
                          │   [auto_approve=True]  →  approve all decisions
                          │   [auto_approve=False] →  mount ApprovalMenu widget
                          │                            await user decision
                          │
                          ├── build resume_payload = {id: {decisions:[...]}}
                          └── stream_input = Command(resume=resume_payload)
                              → continue while True (re-enter stream loop)
        │
        └── [no interrupt_occurred]  →  break while True
              return SessionStats
```

### 2f. Queue draining

When `_cleanup_agent_task()` completes, it calls `_process_next_from_queue()`:

```text
_process_next_from_queue()  [app.py:3350]
  │
  ├── pop QueuedMessage from _pending_messages (FIFO deque)
  ├── remove QueuedUserMessage widget from DOM
  ├── await _process_message(msg.text, msg.mode)
  └── [not busy and more messages]  →  recurse
```

---

## 3. Non-Interactive Scheduling Loop

### 3a. Entry

```text
run_non_interactive(message, ...)  [non_interactive.py:746]
  │
  ├── server_session()  →  start langgraph dev subprocess → RemoteAgent
  ├── build_stream_config(thread_id, assistant_id)
  └── _run_agent_loop(agent, message, config, ...)  [non_interactive.py:604]
```

### 3b. `_run_agent_loop`

```text
_run_agent_loop(agent, message, config, ...)  [non_interactive.py:604]
  │
  ├── StreamState(quiet, stream, spinner)
  ├── stream_input = {"messages": [{"role": "user", "content": message}]}
  │
  ├── await _stream_agent(agent, stream_input, config, state, ...)
  │       async for chunk in agent.astream(
  │         stream_mode=["messages", "updates"],
  │         subgraphs=True, durability="exit",
  │       ):
  │         _process_stream_chunk(chunk, state, console, file_op_tracker)
  │             ├── updates + __interrupt__  →  _process_interrupts() → state.pending_interrupts
  │             └── messages                 →  write to stdout / buffer / spinner
  │
  └── while state.interrupt_occurred:         ← HITL continuation loop
        iterations += 1
        [iterations > MAX_HITL_ITERATIONS]  →  raise HITLIterationLimitError
        │
        ├── _process_hitl_interrupts(state, console)
        │     iterate pending_interrupts
        │     [auto_approve or allow_list match]  →  ApproveDecision
        │     [otherwise]                         →  RejectDecision (fail-closed)
        │     build state.hitl_response
        │
        ├── stream_input = Command(resume=state.hitl_response)
        └── await _stream_agent(agent, stream_input, ...)
```

Key difference from TUI: HITL handling is fully automatic (no interactive
widget). The `--auto-approve` flag controls whether side-effecting commands
are approved or rejected. With `--shell-allow-list`, `ShellAllowListMiddleware`
validates shell commands inline before they reach the interrupt gate, avoiding
a full round-trip.

---

## 4. State Objects During a Turn

| Object | Lives in | Purpose |
| ------ | -------- | ------- |
| `stream_input` | `execute_task_textual` / `_run_agent_loop` local | Initial `{"messages": [...]}` dict or `Command(resume=...)` for HITL continuation |
| `config` | `build_stream_config()` | LangGraph `RunnableConfig` with `thread_id`, `assistant_id`, metadata |
| `session_state.thread_id` | `TextualSessionState` (TUI) | UUID7 identifying the SQLite checkpoint slot |
| `turn_stats` | `SessionStats` | Request count, input/output token totals, wall time |
| `_current_tool_messages` | `TextualUIAdapter` (TUI) | Maps tool-call ID → `ToolCallMessage` widget for in-flight spinner management |
| `pending_text_by_namespace` | `execute_task_textual` local | Accumulated text chunks per `(namespace,)` tuple — prevents subagent text from interleaving with main agent |
| `StreamState` | `_run_agent_loop` local (non-interactive) | Aggregates pending interrupts, HITL responses, full response buffer |

---

## 5. Message Queue (TUI Only)

While `_agent_running` or `_connecting`, incoming chat input is held in a
`deque[QueuedMessage]` (`_pending_messages`) with a matching `QueuedUserMessage`
widget shown inline. Slash commands with a qualifying `bypass_tier` can skip
this queue via `_can_bypass_queue()`.

| `bypass_tier` | Behavior |
| ------------- | -------- |
| `ALWAYS` | Fires unconditionally (e.g. `/quit`) |
| `CONNECTING` | Bypasses only during server startup |
| `IMMEDIATE_UI` | Opens modal immediately; real work deferred |
| `SIDE_EFFECT_FREE` | Takes effect now; chat output waits for idle |
| `QUEUED` | Must wait for the agent to finish |

---

## 6. HITL Pattern Detail

### Interactive TUI

```text
agent.astream(...)
  │  stream_mode="updates"
  │  data["__interrupt__"]   ← graph pauses here
  ▼
pending_interrupts / pending_ask_user collected
  │
  └── [after inner for loop]
        adapter._request_hitl(interrupts)
          └── mount ApprovalMenu / AskUserMenu widget
                user clicks → event fires
                  → set Future result
                      → await future
                          → Command(resume={id: {decisions:[...]}})
                              → agent.astream(Command(...))   ← resumes graph
```

### Non-interactive

```text
agent.astream(...)
  │  __interrupt__ → state.pending_interrupts
  ▼
_process_hitl_interrupts(state, console)
  │  auto_approve=True   → ApproveDecision for all
  │  allow_list match    → ApproveDecision inline (no full HITL round-trip)
  │  otherwise           → RejectDecision (fail-closed)
  ▼
Command(resume=state.hitl_response) → agent.astream(...)
```

---

## 7. Full Control Flow (TUI Path)

```text
cli_main()                           [main.py]
  └── run_textual_cli_async()
        └── DeepAgentsApp.run_async()
              ├── _start_server_background()           [worker]
              │     start_server_and_get_agent()       [server_manager.py]
              │       ├── scaffold langgraph.json workspace
              │       ├── spawn `langgraph dev` subprocess
              │       └── connect RemoteAgent via langgraph-sdk HTTP
              │     post ServerReady → self._agent, self._backend set
              │
              ├── _load_model_background()             [worker]
              │     create_model() → apply_to_settings()
              │
              └── [user types + Enter]
                    on_chat_input_submitted()          [app.py:2228]
                      └── _process_message(value, mode)
                            └── _handle_user_message(message)
                                  ├── mount UserMessage widget
                                  └── _send_to_agent(message)
                                        └── run_worker(
                                              _run_agent_task(message)  [app.py:3284]
                                                └── execute_task_textual(...)
                                                      [textual_adapter.py:355]
                                                      └── while True:
                                                            async for chunk in
                                                              agent.astream(...)
                                                              ├── updates → HITL detect
                                                              └── messages → render
                                                            [interrupt] →
                                                              ApprovalMenu/AskUserMenu
                                                              Command(resume=...)
                                                              (re-enter while loop)
                                                            [done] →
                                                              return SessionStats
                                            )
                                  _cleanup_agent_task()
                                    └── _process_next_from_queue()
```

---

## 8. Comparison to Claude Code QueryEngine

| Dimension | deepagents CLI | Claude Code QueryEngine |
| --------- | -------------- | ----------------------- |
| **Concurrency model** | Textual background `@work` worker; main event loop stays free for key/scroll events while agent streams. | Single async generator in `submitMessage()`; caller `await`s the generator drive loop. |
| **HITL pattern** | Graph pauses at `interrupt_on` tools; client sends `Command(resume=...)` to re-enter. Each approval is a separate `astream()` call and its own LangSmith trace segment. | Inline gate: `canUseTool → wrappedCanUseTool`. Loop never pauses; denials recorded in `permissionDenials` array and continuation is immediate. |
| **Subagent filtering** | Stream namespaces: `ns_key == ()` is main agent; non-empty namespaces are subgraph subagents — filtered from chat view. | Single-agent; no namespace concept. |
| **Message queue** | `deque[QueuedMessage]` + bypass-tier system; slash commands can skip the queue based on tier. | No queue — single `submitMessage()` invocation per user turn. |
| **Streaming render** | Two-mode stream (`"messages"` + `"updates"`); text chunks appended to live `AssistantMessage` widget via `MarkdownStream`. | Typed events (`message_start/delta/stop`); caller renders via event subtype. |
| **Token tracking** | `usage_metadata` captured per chunk; `_context_tokens` persisted in SQLite checkpoint via `TokenStateMiddleware`. | `totalUsage` accumulated in mutable session var; `maxBudgetUsd` enforced as hard cap. |
| **Compact / offload** | `compact_conversation` tool interrupt + `SummarizationToolMiddleware`; chunks tagged `lc_source="summarization"` are hidden, spinner shows "Offloading". | `compact_boundary` / snip replay: older messages trimmed inline; compressed state replayed into new request. |
| **HITL iteration safety** | No explicit cap in TUI (user is always present); non-interactive enforces `_MAX_HITL_ITERATIONS` limit, raises `HITLIterationLimitError`. | `maxTurns` global cap enforced in the main loop; exits with `error_max_turns` result subtype. |
