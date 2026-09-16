# Model-facing time awareness

The agent puts measured wall-clock timings into the model's own context, so the model learns how long operations actually take on this machine instead of relying on assumptions. Each turn's submitted prompt ends with a timing line (session elapsed, duration of the last tool call, wall time since the model's previous reply). Each tool result is stamped with the start, end and duration of the call that produced it. Every number is a real clock reading; a reading that does not exist is reported as `n/a` and never invented. The feature is on by default and, like token awareness, adds no extra messages.

## Behaviour
- Configuration comes from two agent spec fields. A programmatic caller can override each one per session, and an explicit override wins over the spec field, which wins over the default:
  - `time_awareness_enabled` (default `true`): master switch. When false, there is no timing line, no tool stamp and no system-prompt paragraph.
  - `time_awareness_tool_timestamps` (default `true`): stamp tool results. When false, the per-turn timing line stays, and so does the last-tool duration it reports.
- System prompt: when enabled, a newly created session appends this paragraph to the system prompt after the token-awareness block (or after whatever block comes last), separated by a blank line:
  ```
  Wall-clock timing is injected into this session: each of your turns opens with a <session_time> line (session elapsed · duration of the last tool call · wall time since your previous message), and every tool result carries a <tool_time> stamp with its ISO-8601 start, end and duration. These numbers are measured on this machine, not estimated — use them to learn how long operations actually take here instead of assuming. Time passing is information, not pressure: nothing expires, no deadline is implied, and a long elapsed time is never a reason to rush, take shortcuts or stop early.
  ```
- Duration format, used everywhere below (negative values count as 0):
  - under 1 s: whole milliseconds, e.g. `340ms`, `0ms`;
  - 1 s up to 10 s: one decimal place, e.g. `9.4s`;
  - 10 s up to 60 s: whole seconds, e.g. `42s`;
  - 60 s up to 1 h: `<m>m<ss>s` with two-digit seconds, e.g. `1m31s`, `14m22s`;
  - 1 h and above: `<h>h<mm>m` with two-digit minutes, e.g. `2h05m`.
  Components below the displayed precision are truncated.
- Timing line: when enabled, each new user prompt the agent submits gets `\n\n<session_time>session elapsed: <E> · last tool: <L> · wall since your previous message: <W></session_time>` appended (an empty prompt becomes the timing line alone). The separator is ` · ` (space, U+00B7, space).
  - `<E>`: time since the session clock's anchor, which is the session start time.
  - `<L>`: duration of the most recent tool call in the session (from any earlier turn), or `n/a` if no tool has run yet.
  - `<W>`: time since the model's most recent final reply was produced, which is the human's think time; `n/a` before the first reply.
  - The line is part of the content the model receives and of the stored transcript. When the prompt is merged into a preceding user message, the prompt including its timing line is what gets merged. The conversation log keeps the typed text verbatim, without the line.
  - A retry that re-sends the existing transcript without a new prompt adds no timing line.
  - Each time a timing line is produced, a `session_time` event is emitted with `elapsed_seconds` (3 decimals), `last_tool_ms` (integer or null) and `since_previous_message_seconds` (3 decimals or null).
- Tool stamp: the span of each tool call is measured around its execution, so it covers every tool the same way, including the shell tool. The span is also measured for built-in outcomes the agent produces in place of running the tool (for example, a user-interaction tool refused in non-interactive mode). When enabled and tool timestamps are on, the result of that call gets `\n\n<tool_time start="<S>" end="<T>" duration="<D>" />` appended. `<S>` and `<T>` are local ISO-8601 timestamps with millisecond precision and UTC offset (e.g. `2026-09-16T14:22:31.120+02:00`), and `<D>` uses the duration format. A measured span is consumed by the next tool result and stamps only that one result.
- Order in a tool result: the tool's own output, then the `<tool_time …/>` stamp, then any pending token-awareness countdown.
- Every tool call's `tool_result` event and log record gain `started_at` and `ended_at` (both in the stamp's ISO-8601 format) and `duration_ms` (integer, rounded), whatever the knobs say. The last-tool duration is updated whatever the knobs say.
- Snapshot: the message snapshot `metadata` gains `time_awareness`, an object with `enabled` (boolean), `tool_timestamps` (boolean) and `started_at` (the clock anchor as an ISO-8601 timestamp in the stamp format, or null).
- Resume: the session clock keeps its original anchor, so a session resumed a day later reports about a day of elapsed time. A restored session that predates this feature gets the defaults: enabled, tool timestamps on, anchor = its recorded session start time (or now if that is missing or unparseable), no last-tool duration and no previous reply. Explicit overrides given at restore time replace the stored values. The stored system prompt is not rewritten.

## Edge cases
- First turn, no tool run yet: `<session_time>session elapsed: 0ms · last tool: n/a · wall since your previous message: n/a</session_time>`, placed after the typed text and a blank line.
- A turn in which a tool took 340 ms, followed by a 91 s gap before the next prompt: the second prompt reports `last tool: 340ms · wall since your previous message: 1m31s`.
- A tool call whose arguments are malformed, or which is denied before running, gets no `<tool_time>` stamp. The next real call's stamp is not moved onto it.
- The final plain-text reply is never stamped.
- Session restored with a start time two hours in the past: the next timing line reports `session elapsed: 2h00m`.
- `time_awareness_enabled` false: user messages equal the typed text exactly, and no `<session_time>` or `<tool_time` appears anywhere in the transcript.
