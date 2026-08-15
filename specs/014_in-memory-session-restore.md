# In-memory session restore

A library caller can hand a session object that session creation returned earlier back to session creation, and get a live session that continues from that state. Unlike resume by session id, this does not read a snapshot from disk. The restored session keeps its conversation, usage accounting and wiring, gets a fresh log file, and starts with clean compaction state. Some settings can be overridden while restoring.

## Behaviour
- Session creation accepts an optional previously returned session. When one is given, session creation restores it and does not build a new session.
- Before restoring, the given session is checked for every state field a newly created session carries: messages, tool definitions, delivery mode, tool dispatch, session id, cache key, model, endpoint, non-interactive flag, usage totals, provider, output-token cap, strict cache proof flag, model-call count, event handler and subscriptions, streaming flag, options, session start time, and all compaction settings plus the compaction watermark. If any are missing, restoring fails with the error `Session dict is missing required keys: <names>`, where the missing field names are sorted and joined with `, `.
- The restored session keeps these from the given session: messages, usage totals, model-call count, session id, cache key, tool definitions, tool dispatch, event handler and per-event subscriptions, and compaction configuration. The agent spec passed to session creation is not used to rebuild tools or dispatch.
- The restored session is a new top-level object. Replacing its fields does not change the session object the caller passed in.
- A new log file is opened for the restored session, named with the kept session id, following the usual log path layout. The restored session logs to this file, not to the original session's file.
- The compaction watermark (the last observed prompt-token count) is reset to 0.
- These caller arguments override the restored values when supplied (not null): event handler, cache key, output-token cap, tool executor, `compaction_enabled`, `compaction_trigger_tokens`, `compaction_target_tokens`, `compaction_keep_last_turns`, `compaction_policy`, `compaction_min_chars`, `min_cacheable_tokens`. Arguments left unset keep the restored values.
- On restore, a `session_restored` log record is written with `session_id` and `ts` (ISO seconds). A `session_restored` event with data field `session_id` is also emitted, with the display string `Restored session <session id> (<number of messages> messages)`, dimmed.

## Edge cases
- Other session-creation arguments, including non-interactive mode, resume-from id, system-prompt supplement and strict cache proof, have no effect when restoring. The system prompt and the loaded history come from the given session's messages as they are.
- Messages are not re-flattened or reloaded. Tool calls and `tool` messages in the given history are kept exactly as they were.
- A given session that has all required fields plus extra fields (for example a tool executor) keeps the extra fields unless they are overridden.
