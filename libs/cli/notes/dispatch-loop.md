# deepagents-cli Dispatch Loop

This document traces the complete path a user message travels — from keystroke
to LangGraph execution and back — in the interactive TUI mode.

---

## Overview

The dispatch loop has six nested layers:

```text
1. User input (ChatInput widget)
       │
       ▼
2. Queue / bypass decision (on_chat_input_submitted)
       │
       ▼
3. Message router (_process_message)
       │  shell   → _handle_shell_command → ! subprocess
       │  command → _handle_command → slash-command handlers
       └  normal  → _handle_user_message
                        │
                        ▼
4. Agent dispatch (_send_to_agent + run_worker)
       │
       ▼
5. Streaming execution (execute_task_textual in textual_adapter.py)
   │   while True:
   │     async for chunk in agent.astream(…):
   │       "updates"  → collect interrupt/ask_user
   │       "messages" → stream text/tool-calls to widgets
   │   on interrupt → await HITL decision → Command(resume=…)
   │   on completion → break
       │
       ▼
6. Queue drain (_process_next_from_queue)
```

---

## Layer 1 — `ChatInput` widget (`widgets/chat_input.py`)

`ChatInput` is a Textual widget with three input modes:

| Mode | Prefix | Description |
| --- | --- | --- |
| `normal` | (none) | Regular prompt to the agent |
| `shell` | `!` | Direct shell command, bypasses agent |
| `command` | `/` | Slash command |

When the user presses Enter the widget emits a `ChatInput.Submitted` message
carrying `(value, mode)`.

---

## Layer 2 — Queue / bypass decision (`app.py:on_chat_input_submitted`)

```python
async def on_chat_input_submitted(self, event: ChatInput.Submitted) -> None:
```

Decision tree:

1. `/quit` or `/q` always execute immediately (`ALWAYS_IMMEDIATE`).
2. If `_thread_switching` is True, drop the message with a warning toast.
3. If `_agent_running or _shell_running or _connecting`:
   - If the command qualifies for queue bypass (`_can_bypass_queue`), execute
     immediately.
   - Otherwise, append to `_pending_messages` deque and mount a
     `QueuedUserMessage` ghost bubble.
4. Otherwise, call `_process_message(value, mode)` directly.

### Queue bypass tiers

`_can_bypass_queue` inspects the slash command's `BypassTier`:

- `CONNECTING` — allowed while `_connecting and not (_agent_running or _shell_running)`.
- `IMMEDIATE_UI` — only the bare form (no arguments) bypasses.
- `SIDE_EFFECT_FREE` — always bypasses.

---

## Layer 3 — Message router (`app.py:_process_message`)

```python
async def _process_message(self, value: str, mode: InputMode) -> None:
```

Routes by mode:

| Mode | Handler |
| --- | --- |
| `"shell"` | `_handle_shell_command(value.removeprefix("!"))` |
| `"command"` | `_handle_command(value)` |
| `"normal"` (default) | `_handle_user_message(value)` |

### Shell branch (`_handle_shell_command` → `_run_shell_task`)

- Sets `_shell_running = True`.
- Spawns `asyncio.subprocess.Process` via `run_worker(_run_shell_task(…))`.
- Streams stdout/stderr as `ToolCallMessage` output.
- Cleanup in `_cleanup_shell_task` resets `_shell_running` and calls
  `_process_next_from_queue`.

### Command branch (`_handle_command`)

Dispatches to the appropriate handler for each slash command (e.g.,
`_handle_model_command`, `_handle_threads_command`, `_handle_offload`).
Most command handlers complete synchronously, so they do not set
`_agent_running` and the queue drain fires immediately afterward.

### Normal branch

```python
async def _handle_user_message(self, message: str) -> None:
    await self._mount_message(UserMessage(message))   # render bubble
    await self._send_to_agent(message)                # start execution
```

---

## Layer 4 — Agent dispatch (`app.py:_send_to_agent`)

```python
async def _send_to_agent(
    self,
    message: str,
    *,
    message_kwargs: dict[str, Any] | None = None,
) -> None:
```

1. Anchors the chat scroll to the bottom.
2. Sets `_agent_running = True` and disables the `ChatInput` cursor.
3. Calls `self.run_worker(self._run_agent_task(message, …), exclusive=False)`,
   storing the worker handle in `_agent_worker`.

`run_worker` is Textual's non-blocking worker mechanism. The coroutine runs
in the same asyncio event loop but yields control back to the event loop
between `await` points, keeping the UI responsive.

---

## Layer 5 — Streaming execution (`textual_adapter.py:execute_task_textual`)

This is the heart of the dispatch loop.

```python
async def execute_task_textual(
    user_input: str,
    agent: Any,          # RemoteAgent or Pregel
    …
) -> SessionStats:
```

### Outer `while True` loop

The outer loop exists to handle HITL (human-in-the-loop) resume cycles. Each
iteration drives the agent forward until it either completes or pauses for
user input:

```python
while True:
    interrupt_occurred = False
    pending_interrupts: dict[str, HITLRequest] = {}
    pending_ask_user: dict[str, AskUserRequest] = {}

    async for chunk in agent.astream(
        stream_input,              # {"messages": [user_msg]} or Command(resume=…)
        stream_mode=["messages", "updates"],
        subgraphs=True,
        config=config,             # thread_id, assistant_id, sandbox_type
        context=context,           # model override, model params
        durability="exit",
    ):
        namespace, current_stream_mode, data = chunk

        if current_stream_mode == "updates":
            # Collect interrupts and ask_user requests
            …
        elif current_stream_mode == "messages":
            # Render content and tool calls
            …

    # After stream exhausted:
    if interrupt_occurred and resume_payload:
        stream_input = Command(resume=resume_payload)   # resume cycle
    else:
        await dispatch_hook("task.complete", …)
        break                                           # done
```

### Inner `async for chunk` loop — stream modes

LangGraph streams 3-tuples `(namespace, mode, data)` when `subgraphs=True`.

**`"updates"` stream** — fires once per graph node completion:

- `__interrupt__` key → one or more `Interrupt` objects.
  - `type == "ask_user"` → validated as `AskUserRequest`, added to
    `pending_ask_user`.
  - Other → validated as `HITLRequest` (tool approval), added to
    `pending_interrupts`.

**`"messages"` stream** — fires once per token / message chunk:

- `HumanMessage` chunk — flush any pending assistant text.
- `AIMessage` / `AIMessageChunk`:
  - Text delta → accumulate in `pending_text_by_namespace[ns_key]`, then flush
    to a streaming `AssistantMessage` widget.
  - Tool call delta → accumulate in `tool_call_buffers`, create
    `ToolCallMessage` widget when the call is complete.
- `ToolMessage` → call `tool_msg.set_result(content)` to update the widget.

Subagent namespaces (non-empty `ns_key`) are filtered out — only main-agent
output is rendered in the chat.

### HITL handling (after the inner loop)

For each pending interrupt:

1. Hide spinner, show `ApprovalMenu` or `AskUserMenu` widget.
2. `await future` — blocks until the user decides.
3. Map the UI decision to a LangGraph `Command(resume={id: decisions})`.

If any interrupt is rejected, mount a "Command rejected" message and return
early without resuming. Otherwise set `stream_input = Command(resume=…)` and
continue the outer loop.

### Token accounting

`_is_summarization_chunk(metadata)` detects model output tagged
`lc_source="summarization"`. Those chunks switch the spinner to "Offloading"
and are not rendered.

At end of stream, `_report_and_persist_tokens` reads final token counts from
the graph state and updates the `StatusBar` display.

---

## Layer 6 — Queue drain (`app.py:_process_next_from_queue`)

Called at the end of `_cleanup_agent_task` (and `_cleanup_shell_task`):

```python
async def _process_next_from_queue(self) -> None:
    if self._processing_pending or not self._pending_messages or self._exit:
        return

    self._processing_pending = True
    try:
        msg = self._pending_messages.popleft()
        if self._queued_widgets:
            widget = self._queued_widgets.popleft()
            await widget.remove()          # remove ghost bubble
        await self._process_message(msg.text, msg.mode)
    finally:
        self._processing_pending = False

    # If it was a synchronous command (no worker spawned), drain again
    busy = self._agent_running or self._shell_running
    if not busy and self._pending_messages:
        await self._process_next_from_queue()
```

Design points:

- `_processing_pending` prevents re-entrant drains (two concurrent calls
  would corrupt the deque).
- Ghost `QueuedUserMessage` widgets are removed before processing so the UI
  stays in sync.
- Slash commands complete synchronously; the recursive tail call continues
  draining without waiting for a new worker finish event.

---

## Complete call chain (happy path)

```text
cli_main()
  └─ asyncio.run(run_textual_cli_async(…))
       └─ run_textual_app(…)
            └─ DeepAgentsApp.run()
                 └─ on_mount → _post_paint_init
                      └─ _start_server_background (worker)
                           └─ ServerReady event → on_deep_agents_app_server_ready
                                └─ set _agent, _ui_adapter

  [user types a message and presses Enter]
  ChatInput.Submitted event
    └─ on_chat_input_submitted(event)
         └─ _process_message(value, "normal")
              └─ _handle_user_message(message)
                   ├─ mount UserMessage widget
                   └─ _send_to_agent(message)
                        └─ run_worker(_run_agent_task(message))  ← Textual worker
                             └─ execute_task_textual(…)           ← textual_adapter.py
                                  └─ while True:
                                       async for chunk in agent.astream(…):
                                         "messages" → stream to AssistantMessage widget
                                         "updates"  → collect HITL interrupts
                                       [interrupt?] → ApprovalMenu → Command(resume)
                                       [done]       → break
                             └─ _cleanup_agent_task()
                                  └─ _process_next_from_queue()   ← drain pending
```

---

## Key files

| File | Responsibility |
| --- | --- |
| `app.py` | `DeepAgentsApp`, input handling, queue, HITL wiring |
| `textual_adapter.py` | `execute_task_textual`, streaming loop, UI callbacks |
| `widgets/chat_input.py` | `ChatInput`, multi-mode input widget |
| `widgets/messages.py` | `AssistantMessage`, streaming markdown rendering |
| `widgets/approval.py` | `ApprovalMenu`, HITL decision widget |
| `command_registry.py` | `SlashCommand`, `BypassTier` |
| `agent.py` | `create_cli_agent`, middleware stack |
| `server_manager.py` | LangGraph server subprocess lifecycle |
| `remote_client.py` | `RemoteAgent` wrapping `langgraph_sdk.RemoteGraph` |
