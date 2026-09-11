# Session directories and durable sinks

A library caller can give one session an explicit directory that holds its journal, snapshot and event log, independent of the model name and the working directory. A caller can also attach a durable sink: a synchronous destination that receives the session's ordered lifecycle records. With either one, the agent captures the whole lifecycle (system prompt, messages, model requests and responses, streaming frames, tool boundaries, runtime events) and commits each record before it is sent to a model, a tool, a renderer or a subscriber.

## Behaviour

### Options
- Every library entry point that creates a session (session creation, run task, the one-shot direct-tool agent, the backward-compatible run helper, the synchronous and asynchronous REPLs) accepts an optional session directory and an optional durable sink. Both are off by default. Without them, behaviour and file layout are unchanged.
- Durable capture is on when a session directory or a sink is given. For an in-memory restore, a directory or sink passed on restore replaces the saved one, and capture is on if the restored session has either one.

### Session directory
- The directory is created if missing. It holds:
  - `journal.jsonl`: the durable journal (same format as the default journal).
  - `messages.json`: the conversation snapshot (same format as the default snapshot).
  - `events.jsonl`: the session log (same record format as the default log).
- A session directory forces durability on, even when the caller or the agent spec turns it off.
- A session directory has a single writer. Opening its journal takes an exclusive, non-blocking lock on `journal.jsonl.lock` in the same directory, created with mode `0600`. If another writer holds the lock, session creation fails with the error `session journal is already open: <journal path>`. The lock is released when run task finishes (success or error) and when a REPL exits.
- Resuming by session id with a session directory:
  - The snapshot is read from `<dir>/messages.json`. It is either an object whose `messages` field is the history, or a bare array. It is normalised like any resumed snapshot. A missing file, or a history that is not an array, loads nothing.
  - The journal replayed is `<dir>/journal.jsonl`.
  - No cross-model snapshot port and no endpoint rebinding happen. The caller's schema is used as given.

### Durable capture
- With durable capture on, every log line is flushed and fsync'd before the agent continues. Every snapshot write is flushed and fsync'd, then its directory is fsync'd.
- Lifecycle records are appended to the journal (when the session has one) and then handed to the sink. Besides the journal record types, the stream contains:
  - `message` with `msg`: the system prompt, written at session creation before the session can send it or report any startup event.
  - `event` with `event_type` and `data` (the event's data fields): written before any per-event handler, general event handler or the terminal renderer sees the event.
  - `model_request` with `request`: the exact request parameters for a model call, excluding callbacks and authentication state. Written before the call is made.
  - `model_request_attempt` with `attempt` = `{"payload": <request body>}`: written before each attempt to send the request, including retries.
  - `model_response_frame` with `frame`, written as the response arrives: `{"kind": "payload", "payload": <parsed JSON body>}` for a non-streaming HTTP response, `{"kind": "sse", "line": <line>}` for every streamed line (blank lines included), and `{"kind": "subprocess", "stdout": ..., "stderr": ...}` for a subprocess-backed model.
  - `model_response` with `response` = `{"content", "reasoning", "tool_calls"}`, each tool call as `{"id", "type", "name", "arguments"}`. Written after the call returns, before the agent acts on it.
  - For the pre-compaction preservation call: the same pair with `purpose: "pre_compaction"` and the same `request` and `response` shapes as below. The `model_response` record is written only when the call succeeds.
  - For context compaction: a `model_request` with `purpose: "compaction"` and `request` = `{"model", "messages", "temperature": 0, "max_tokens": <compaction target tokens>}`, and a `model_response` with `purpose: "compaction"` and `response` = `{"content"}`.
- Journal recovery records (`turn_start`, `turn_end`, `message`, `tool_start`, `tool_end`, `reset_messages`) are also mirrored to the sink.

### Durable sink
- A sink is any object with a synchronous append operation that takes one record (a JSON-shaped object). It must not return until the record is durable. Queueing a background write does not meet the contract.
- The sink gets a copy of each record, in stream order, after the built-in journal has committed it and before any downstream consumer (event handler, model request, tool dispatch) uses it.
- An error raised by the sink propagates and stops the operation that produced the record. For example, the event is not delivered, or the model call or tool is not run.

## Edge cases
- `model_request_attempt` and `model_response_frame` records come only from the built-in HTTP and subprocess clients. A caller-supplied client produces `model_request` and `model_response` only.
- Replay counts lifecycle records as replayed entries (unknown types). The journaled system prompt `message` becomes part of the rebuilt history.
- A sink with durability off and no session directory still gets the lifecycle records. It gets no turn or tool records, because they are only written when a journal exists.
