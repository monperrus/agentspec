# Side query

A library caller can ask the model a question about a session while a turn is still running (for example from another thread while the turn is blocked in a long tool call) and get a short text answer, without altering the session's history or journal (and, by default, its usage totals).

## Behaviour
- A side query takes a client, a model, a session and a question, plus three optional settings: an output-token cap, a preamble (default: the one below; none sends the question alone) and a usage-accounting switch (default off). It returns the model's answer text with leading and trailing whitespace removed (empty string when the reply has no content).
- It works on a copy of the session's current messages; the session's messages are never modified, and nothing is added to the durable journal or the model-call count. Nothing is added to the usage totals unless usage accounting is on.
- In-flight tool calls: each tool call of the last assistant message in the copy that has no `tool` message with its id yet gets a `tool` message appended after the existing messages, with that `tool_call_id` and content `[still running: this tool call has not returned yet]`. Calls that already have results keep them unchanged.
- The copy then gets the same tool-call pairing repair as a resumed history (placeholders for any other unanswered id, orphan `tool` messages dropped).
- The question is sent as user content prefixed by the preamble and a blank line (`\n\n`). The default preamble is:
  `[Side question from the user while your turn is still running. Answer it briefly from the conversation so far; you cannot call tools here. Your running work continues untouched afterwards, and this exchange is not kept in your context.]`
  With no (or an empty) preamble, the content is the question alone.
- In structured mode the content is appended as a new user message. In non-structured mode, when the last message is a user message (e.g. inline tool results), the content is appended to that message's content after a blank line instead, so user/assistant alternation is kept; otherwise a new user message is appended.
- Exactly one model request is made, with the same request parameters as a turn so the cached prompt prefix is reused: temperature 0, the cache key as `user` and (unless excluded) as `prompt_cache_key`, and `max_tokens` set to the side query's own cap when given, else to the session's output-token cap when one is configured. In structured mode the session's `tools` are declared unchanged but with `tool_choice: "none"`; in non-structured mode neither `tools` nor `tool_choice` is sent.
- Against an OpenRouter endpoint, the request uses the same routing as a turn: `usage: {"include": true}` and the session's (pinned or configured) `provider`, so the cached prefix is served by the same provider.
- The exchange is recorded only in the session log, as one record `{"type": "side_query", "question": <question>, "answer": <answer>}`, plus a `usage` object (`prompt_tokens`, `completion_tokens`, `total_tokens`, `cached_tokens`, `cache_creation_tokens`, missing values as 0) when the response reported usage.
- With usage accounting on and usage reported, those counts are added to the session's usage totals (prompt, completion, total, cached, cache write) and a `usage` event is emitted with `purpose: "side_query"`, the usual `prompt`, `completion`, `total`, `cached`, `cache_write` fields, and a `fmt` line prefixed `[tokens · side query]` (showing `prompt ?` when the agent spec declares `reports_prompt_tokens: false`).

## Edge cases
- API errors from the request propagate to the caller; nothing is logged in that case.
- Only the trailing assistant message's unanswered calls are treated as in flight; unanswered calls of earlier assistant messages get the usual "tool result lost" placeholders from the pairing repair.
- Tool calls without an `id` get no in-flight result.
- With usage accounting on but no usage in the response, nothing is accounted and no `usage` event is emitted.
- Any tool calls in the model's reply are ignored; they are never executed.
