# Cross-model session port

Resuming a session with a different model than the one it was created with (`<program> <other-model> --session <id>`) is a deliberate provider switch. Instead of replaying the old provider's snapshot in place, the agent copies the snapshot into the new model's directory and re-stamps its metadata for the new provider. The ported file then behaves like a native session of the new model, so later resumes stay on the new provider, and the original file is kept as the record of where the history came from.

## Behaviour
- Porting happens whenever a session is initialised with a resumed session id (CLI, library one-shot or REPL run, or a direct session initialisation), before the endpoint rebinding of a resumed session.
- When the current model already has a snapshot `<safe model>/<id>_messages.json`, nothing is ported and that snapshot is used unchanged (idempotent: a second resume does not re-copy).
- Otherwise, among the snapshots `<other safe model>/<id>_messages.json` under other model directories, the most recently modified one is the source.
- The source must be a JSON object whose `messages` is an array. The copy keeps `messages` and every other top-level key unchanged, and replaces `metadata` with:
  - `model`: the new model name as given.
  - `endpoint`: the endpoint the new run resolved (`""` if none).
  - `session_id`: the resumed id.
  - `ported_from`: `{"model": <source metadata.model>, "endpoint": <source metadata.endpoint>}`, each `null` when the source lacks it.
  - `default_tools`, `tools` and the tool-provenance commit key, copied from the source metadata when present there.
  - `auth`: the new run's key-source configuration: whichever of `auth`, `keyring_service`, `keyring_username` and `key_env` have a non-null value, copied as is (`{}` when none). Never a secret value.
- The copy is written to the new model's directory (created if missing) as UTF-8 JSON with 2-space indent. The source file is never modified.
- After porting, a one-line notice is printed: `Ported session <id> from <source safe model dir> to <new safe model>; continuing on <endpoint>`, where `<endpoint>` is `the new endpoint` when the resolved endpoint is empty.
- The resumed run then loads the ported snapshot as the current model's own. The endpoint rebinding finds the ported metadata, so the run continues on the new model's endpoint and key source, and every snapshot it saves records the new endpoint.
- A resume with the session's own model still rebinds to the session's original endpoint as before; porting does not affect it.

## Edge cases
- No snapshot with that id under any model directory (or no log base directory at all): nothing is ported; the resume proceeds as a fresh session under that id.
- A source that is unreadable, not valid JSON, a legacy plain array of messages, or whose `messages` is not an array is not ported. The history may still be loaded from the other model's snapshot, but the endpoint binding does not consult it, so the run stays on the new model's resolved endpoint.
- Source metadata that is missing or not an object is treated as empty: `ported_from` then has `null` fields and no tool provenance is copied.
