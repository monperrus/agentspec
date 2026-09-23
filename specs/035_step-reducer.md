# Per-step context policy

A library caller can attach a step reducer to a session: a callback run after every tool step that decides what of that step stays in the history. Combined with a side query, this lets the model replace raw tool output by a bounded digest of its own step, so the context grows by at most a fixed number of tokens per step. The same capability also exposes the full tool runtime to hosts that run their own agent loop on a session.

## Behaviour
- A step reducer is optional (default: none) and is accepted wherever a session is created: session initialisation, the one-shot task runner, the direct-tool one-shot agent, the backward-compatible runner and both interactive REPLs. With no reducer, behaviour is unchanged.
- A step is the assistant message carrying the tool calls followed by all of its results: in structured mode the assistant message plus its `tool` messages; in inline mode the assistant message plus the `Tool results:` user message.
- After each step is fully appended to the history, the reducer is called with the session, a copy of the step's messages, the client and the model. Its return value can be:
  - nothing → the step is kept unchanged;
  - a list of messages → these replace the step in the history, at the step's position;
  - a reduction (a list of messages plus an optional final reply) → the messages replace the step; when the final reply is set, the turn ends.
- Replacement messages without a `ts` field get the current local time (ISO 8601, seconds) as `ts`.
- The next model request of the turn is built from the reduced history; the raw step is never sent again.
- The replacement must leave the history valid for the next request (end on a `user` or `tool` message); this is the reducer's responsibility and is not checked.
- A final reply ends the turn exactly as if the model had replied that text: it is appended as an assistant message after the replacement messages, becomes the turn's final reply, no further model request is made, and turn-end processing (Stop hooks, turn-end compaction, final-answer event) runs as usual.
- Each reduction, when the durable journal is active, appends a `reset_messages` record with `reason` `step_reducer` and the full reduced history, so recovery replays the reduced history rather than the raw step.
- Each reduction is logged in the session log as `{"type": "step_reduced", "raw_messages", "kept_messages", "raw_chars", "kept_chars", "final", "ts"}` (`final` is a boolean) and emits a `step_reduced` event with `raw_messages`, `kept_messages` (message counts), `raw_chars`, `kept_chars` (total length of the messages serialised as JSON), `final_reply` (text or null) and `fmt` `[step reduced] <n> msgs/<c> chars → <m> msgs/<k> chars` (character counts with thousands separators).
- The reducer is runtime-only: it is not part of the session snapshot. When a session is restored with a reducer supplied, that reducer is attached; otherwise the restored session has none.
- Executing a single tool call: a host running its own loop can execute one named tool call with arguments (and an optional call id) on a session through the same path the turn loop uses — pre/post tool-use hooks, the durable journal, `tool_call`/`tool_result` events, the session's tool executor (sandbox) and time awareness. It returns the model-facing result text (`ERROR: ...` on failure). This differs from plain dispatch, which only invokes the tool function.

## Edge cases
- A reducer returning nothing produces no journal record, log record or event.
- A step whose model reply also contained text is still reduced; with a final reply, the reducer's text replaces the model's text as the turn's answer.
- An empty replacement list removes the step entirely.
