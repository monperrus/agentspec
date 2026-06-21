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
  - `tool_result` (after a tool returns): `name`, `result`, `streamed`.
  - `content_delta` / `reasoning_delta` (streamed text or reasoning chunk): `text`, `first`, `no_newline`.
  - `content_stream_end` / `reasoning_stream_end` (end of a streamed sequence): `no_newline`.
  - `usage` (per-turn token usage) and `session_usage` (cumulative, emitted with the final answer): `prompt`, `completion`, `total`, `cached`, `cache_write`.
  - `error` (API or dispatch error): `text`.
  - `final_answer` (the agent's final reply): `text`.
  - `token_limit` (token budget exceeded): `used`, `limit`.
  - `session_resumed` (history loaded, or not found): `session_id`, `messages_loaded`, and optionally `source_model`.
  - `provider_pinned` (OpenRouter provider locked for the session): `provider`.

## Edge cases
- Emitting an event type with no subscribed handlers is not an error; only the generic handler runs.
- Unsubscribing a handler that was never subscribed, or for an event type never used, does nothing and raises no error.
- After unsubscribing, the handler is no longer called; other handlers for that type are unaffected.
- Subscribing to an event type name that the agent never emits is accepted; the handler is simply never called.
