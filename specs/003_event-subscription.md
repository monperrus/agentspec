# Event subscription

A caller embedding the agent as a library can subscribe handlers to individual event types on a session, in addition to the single generic event handler, so that logging, dashboards or UIs can react to specific events. Each event has a documented, stable set of data fields.

## Behaviour
- A caller can subscribe a handler to one event type on a given session, and unsubscribe it later. A second name for subscribing exists as a pure alias with identical behaviour.
- A handler receives the event type and the event's data fields.
- Several handlers may be subscribed to the same event type; on each event they are called in subscription order.
- When an event fires, all handlers subscribed to its type are called first, then the generic handler supplied at session creation (or the default printing handler if none was supplied). The generic handler still receives every event.
- Subscriptions are per session and per event type: a handler subscribed to one type is never called for another type or another session.
- Every event's data includes `fmt`, a pre-formatted ANSI display string (what the default handler prints). Other fields per event type:
  - `tool_call` (before a tool is dispatched): `name`, `args`.
  - `tool_result` (after a tool returns): `name`, `result`, `streamed`, `files`, `diff_summary`.
    - `files`: list of file paths created or modified by the call, exactly as passed by the model (e.g. `["src/main.py"]`); null for tools that do not write files.
    - `diff_summary`: object `{path, added, removed}` with line counts, so consumers can display `+5 -2 src/main.py` without re-reading the file; null for tools that do not write files.
    - Built-in whole-file write: `added` = number of lines in the written content, a trailing newline not starting an extra line (`"a\nb\n"` → 2; empty content → 0), `removed` = 0.
    - Built-in substring replacement: `removed` = line count of the old text, `added` = line count of the new text, where a non-empty text has (number of newlines + 1) lines and an empty text has 0, each multiplied by the number of occurrences actually replaced (1 by default, all of them with `replace_all`). E.g. replacing `1` by `2` in `x=1\ny=1\nz=1\n` gives `added` 1, `removed` 1 by default, and 3/3 with `replace_all`.
  - `content_delta` / `reasoning_delta` (streamed text or reasoning chunk): `text`, `first`, `no_newline`.
  - `content_stream_end` / `reasoning_stream_end` (end of a streamed sequence): `no_newline`.
    - Reasoning deltas always precede content deltas within a turn. When a turn streamed both reasoning and content, `reasoning_stream_end` is emitted before `content_stream_end`, so consumers that buffer reasoning until its end event never lose the trace. In that case its `fmt` is empty (the console's `[thinking]` line was already terminated by the first content delta's leading newline), while `content_stream_end` keeps `fmt` = `"\n"`.
    - When only reasoning streamed (no content), `reasoning_stream_end` is still emitted, and no `content_stream_end` is.
  - `usage` (per-turn token usage) and `session_usage` (cumulative, emitted with the final answer): `prompt`, `completion`, `total`, `cached`, `cache_write`.
  - `error` (API or dispatch error): `text`, `error_class`, `http_status`, `elapsed_s`, `adapter` (see the structured error diagnostics spec).
  - `final_answer` (the agent's final reply): `text`.
  - `token_limit` (token budget exceeded): `used` (effective, non-cached tokens), `limit`, `raw_total` (session raw API total), `cached_total` (session cached tokens).
  - The `session_usage` display line reads `[session tokens] prompt <p>  |  completion <c>`, with numbers using thousands separators (e.g. `1,234`). When the session has cached tokens, `  |  cached <n>` is inserted between the prompt and completion fields. The line shows no total and no effective count; the event data still carries `total`.
  - `session_resumed` (history loaded, or not found): `session_id`, `messages_loaded`, and optionally `source_model`.
  - `session_restored` (a previously returned session object restored in memory): `session_id`.
  - `provider_pinned` (OpenRouter provider locked for the session): `provider`.
  - `compaction` (older context replaced by a summary): `summary`, `compacted_turns`.
  - `cache_cold` (a strict-cache check was waived because the prefix cache expired on a cold resume): `age` (whole seconds since the last timestamped message).
  - `cache_proof_missing` (in strict cache mode, a call after the first showed no cache proof or no cache hit; the turn continues): `cached_tokens` and `prompt_tokens` when a usage block was present.
  - `journal_recovered` (a resumed session was rebuilt from the durable journal): `entries_replayed`, `messages_loaded`, `pending` (number of in-flight tool calls), `mid_turn`.
  - `rate_limit_wait` (a retryable HTTP 429 was received and the agent is about to wait before resending): `delay` (seconds, a number), `resume_at` (ISO 8601 timestamp including the UTC offset, equal to the local time when the wait started plus `delay`). `fmt` is the `[rate-limited] waiting …` line printed to the console for the same wait (without a trailing newline). The event fires once per wait, before the wait starts, for the model calls of a turn (streaming or not) and for compaction summary calls.

## Edge cases
- Emitting an event type with no subscribed handlers is not an error; only the generic handler runs.
- Unsubscribing a handler that was never subscribed, or for an event type never used, does nothing and raises no error.
- After unsubscribing, the handler is no longer called; other handlers for that type are unaffected.
- When a built-in write or replacement fails (the result starts with `ERROR:`), `files` and `diff_summary` are null.
- Read-only tools, shell commands and custom tools that report no file metadata yield null `files` and `diff_summary`.
- `rate_limit_wait` never fires for a 429 without a usable retry delay (that is a rate-limit error, not a wait), for client-side RPM throttling waits, for read-timeout retries, for transient 5xx retries, or for `run://` subprocess backends. The `[rate-limited]` console line is still printed directly, in addition to the event.
- Subscribing to an event type name that the agent never emits is accepted; the handler is simply never called.
