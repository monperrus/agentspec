# Side query

A library caller can ask the model a question about a session while a turn is still running (for example from another thread while the turn is blocked in a long tool call) and get a short text answer, without altering the session's history, journal or usage totals.

## Behaviour
- A side query takes a client, a model, a session and a question, and returns the model's answer text with leading and trailing whitespace removed (empty string when the reply has no content).
- It works on a copy of the session's current messages; the session's messages are never modified, and nothing is added to the durable journal, the usage totals or the model-call count.
- In-flight tool calls: each tool call of the last assistant message in the copy that has no `tool` message with its id yet gets a `tool` message appended after the existing messages, with that `tool_call_id` and content `[still running: this tool call has not returned yet]`. Calls that already have results keep them unchanged.
- The copy then gets the same tool-call pairing repair as a resumed history (placeholders for any other unanswered id, orphan `tool` messages dropped).
- The question is sent as user content prefixed by this preamble and a blank line (`\n\n`):
  `[Side question from the user while your turn is still running. Answer it briefly from the conversation so far; you cannot call tools here. Your running work continues untouched afterwards, and this exchange is not kept in your context.]`
- In structured mode the content is appended as a new user message. In non-structured mode, when the last message is a user message (e.g. inline tool results), the content is appended to that message's content after a blank line instead, so user/assistant alternation is kept; otherwise a new user message is appended.
- Exactly one model request is made, with the same request parameters as a turn so the cached prompt prefix is reused: temperature 0, the cache key as `user` and (unless excluded) as `prompt_cache_key`, and `max_tokens` when an output-token cap is configured. In structured mode the session's `tools` are declared unchanged but with `tool_choice: "none"`; in non-structured mode neither `tools` nor `tool_choice` is sent.
- The exchange is recorded only in the session log, as one record `{"type": "side_query", "question": <question>, "answer": <answer>}`.

## Edge cases
- API errors from the request propagate to the caller; nothing is logged in that case.
- Only the trailing assistant message's unanswered calls are treated as in flight; unanswered calls of earlier assistant messages get the usual "tool result lost" placeholders from the pairing repair.
- Tool calls without an `id` get no in-flight result.
- Any tool calls in the model's reply are ignored; they are never executed.
