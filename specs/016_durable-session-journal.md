# Durable session journal

The conversation snapshot is only written at turn boundaries, so a crash mid-turn would lose the whole turn, including tool calls whose side effects already happened. With durability on (the default), the agent also keeps an append-only write-ahead journal beside the snapshot and flushes every record to stable storage as it happens. When a session is resumed, the journal is replayed and takes precedence over a stale snapshot. Completed tool results the model never saw are handed back to it instead of being re-run, and tool calls that were in flight at the crash are flagged so the model checks state before re-running them.

## Behaviour

### Enabling
- Durability is on by default. The agent spec field `durable` (boolean) sets it. A library caller's explicit setting overrides the spec. The CLI flag `--no-durable` turns it off, and without the flag the spec value applies.
- With durability off, no journal is written and a session behaves exactly as with turn-boundary snapshots only.
- When an in-memory session is restored, an explicit caller setting overrides the restored session's setting, which in turn defaults to on. The restored session reopens (or starts) the journal for its session id and appends to it.

### Journal file
- Path: `~/.local/share/agent_probe/<safe model>/<session id>_journal.jsonl`, in the same directory as the snapshot. The directory is created if missing.
- JSON Lines, UTF-8, non-ASCII characters written as is, one record per line. Each record is appended and flushed to disk (fsync) before the agent continues.
- Every record has `seq`, `ts` and `type`. `seq` is 1-based and strictly increasing. When an existing journal is reopened, numbering continues after the number of non-empty lines already in the file. `ts` is ISO 8601 local time with milliseconds.
- Record types and their extra fields:
  - `turn_start`: `task`, the user's input for the turn. Written when a turn begins.
  - `turn_end`: no extra fields. Written whenever a turn ends: final answer, error or interrupt.
  - `message`: `msg`, the message exactly as it was appended to the history. One record for each user, assistant and `tool` message and each tool-results message added during a turn. When the new user input is merged into a trailing user message, the merged message is written again as a new `message` record.
  - `tool_start`: `call_id`, `name`, `args`. Written before the tool runs.
  - `tool_end`: `call_id`, `result` (the result text returned to the model). Written after the tool returns. A tool that ends the process with a fatal error gets no `tool_end`.
  - `reset_messages`: `reason` and `messages` (the full replacement history). Written when the history is replaced: `reason` is `compaction` after context compaction and `clear` after `/clear`.
- `call_id` is the provider's tool-call id for structured tool calls. For inline tool calls, which have no provider id, it is a fresh random 12-hex-character id per call.

### Replay
- Lines are read in order. Blank lines, lines that are not valid JSON, and JSON values that are not objects are skipped. Every other line counts as a replayed entry, including records of unknown type.
- `message` appends its `msg` (when it is an object) to the rebuilt history. `reset_messages` replaces the rebuilt history with its `messages` (object entries only), or empties it when `messages` is not an array.
- `tool_start` marks the call as pending. A missing `call_id` gets a fresh random id, a missing `name` becomes `?`, and non-object `args` become `{}`. A later `tool_start` with the same id replaces the earlier one.
- `tool_end` clears the pending call with that id, if any, and records its result as known.
- The journal is mid-turn when more `turn_start` than `turn_end` records have been seen. A `turn_end` with no open turn does not make the count negative.
- A known result is unreceived when no `tool` message in the rebuilt history has that `tool_call_id`. Unreceived results are ordered by call id.

### Recovery on resume
- When a session is resumed by id, the snapshot is loaded first, as usual. Then the journal for that session id under the current model is replayed.
- The journal is used when it has at least one replayed entry and at least one of these is true: no messages were loaded from a snapshot, the journal is mid-turn, or the rebuilt history has more than one message. Otherwise the snapshot history is kept unchanged.
- When the journal is used:
  - The session history becomes the journal's rebuilt history, flattened with the same rules as a resumed snapshot (merged same-role messages, tool calls and `tool` messages turned into plain assistant text).
  - A `journal_recovered` log record is written with `resumed_from`, `entries_replayed`, `messages_loaded` (the number of rebuilt messages), `pending_tool_calls` (a list of objects with `call_id` and `name`), `mid_turn` and `ts` (ISO seconds).
  - A `journal_recovered` event is emitted with `entries_replayed`, `messages_loaded`, `pending` (the number of pending calls) and `mid_turn`. The display string is `Recovered <messages loaded> messages from the durable journal (snapshot had <snapshot message count>)`, dimmed.
  - If there are pending calls, a user message (with `ts`) is appended:
    ```
    SYSTEM RECOVERY NOTE: The previous session crashed while these tool calls were in flight; their side effects are UNKNOWN:
    <calls>
    Verify the resulting state (inspect files, re-run read-only checks) before re-running any of them.
    ```
    `<calls>` has one line per pending call, `- <name>(<args as JSON, non-ASCII as is>)`, joined by newlines.
  - Then, if there are unreceived results, another user message (with `ts`) is appended: `SYSTEM RECOVERY NOTE: These tool calls completed just before the crash but their results were never shown to you. Do NOT re-run them; treat these results as observed:\n<results>`. `<results>` has one line per result, `- <label>: <result>`, joined by newlines. `<label>` is the record's tool name when it has one, and otherwise the call id. `tool_end` records carry no name, so in practice the label is the call id.
- Neither pending nor completed tool calls are re-executed by recovery.

## Edge cases
- A torn last line (the process died mid-write) is not valid JSON, so it is ignored. That record was never acknowledged.
- A missing journal file replays to nothing: the resume uses the snapshot exactly as before. This covers sessions from before durability existed and sessions run with durability off.
- The rebuilt history holds only what the journal recorded. Messages that existed before the journal was started (such as the system prompt) are not part of it unless a `reset_messages` record carried them.
- The journal is never truncated. Compaction and `/clear` append a `reset_messages` record, so replay reproduces the replaced history rather than the discarded one.
- A tool call with a `tool_end` whose `tool` message was also journaled counts as received and gets no recovery note. Inline tool calls never produce `tool` messages, so a completed inline call always counts as unreceived.
