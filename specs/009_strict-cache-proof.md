# Strict cache proof

By default the agent runs fail-closed on prompt caching: after the first model call of a session, every later call must return explicit server-side cache accounting showing a nonzero cache hit. If it does not, the turn is aborted with an error. Models that cannot prove cache reuse are treated as unsupported. Callers and the CLI can opt out.

## Behaviour
- Strict mode is on by default for single tasks, the sync REPL and the async REPL. The CLI flag `--no-strict-cache-proof` disables it; programmatic callers can disable it per session.
- The session counts model calls that returned a usage block. The count spans the whole session, across turns; it is not reset per turn.
- Usage normalization, from the response `usage` object (`details` = `usage.prompt_tokens_details`):
  - Cached (cache-read) tokens are taken from the first of these fields that is present (non-null): `details.cached_tokens`, `usage.cache_read_input_tokens`, `usage.cache_read_tokens`, `details.cache_read_tokens`, `details.cache_read_input_tokens`; 0 if none is present.
  - Cache-write tokens are taken from the first present of: `usage.cache_creation_input_tokens`, `usage.cache_write_tokens`, `details.cache_write_tokens`, `details.cache_creation_input_tokens`; 0 if none is present.
  - The response carries cache proof when any of these nine fields is present, even with value 0, and even if only a cache-write field is present.
- These normalized values feed the per-response `usage` event/log record and the session totals (`cached`, `cache_write`) for every response, regardless of strict mode.
- In strict mode, for each response with a usage block, once the session count is 2 or more:
  - No cache-proof field → the turn aborts with the error `Strict cache mode requires explicit cache accounting from the server after the first LLM call, but this response exposed no cache-proof field.`
  - Cached tokens ≤ 0 → the turn aborts with the error `Strict cache mode requires cached_tokens > 0 after the first LLM call, but the server reported no cache hit.`
- In strict mode, a response with no usage block after at least one counted call aborts the turn with the error `Strict cache mode requires usage metadata on every LLM call after the first, but the server returned no usage block.`
- An abort emits an `error` event (displayed as `Error: <text>`), appends an `error` log record, and ends the turn normally with the turn result (session id, the last non-empty assistant reply so far, cumulative usage, messages). It does not crash the REPL or the process.

## Edge cases
- The first counted call is never checked, so a cold cache on the first call is allowed.
- A response without a usage block is not counted; if it is the first response of the session, it passes.
- The failing response is discarded: its usage is not added to totals or emitted, and its tool calls or reply are not processed.
- A present-but-zero field wins over later candidates: `details.cached_tokens: 0` with `usage.cache_read_input_tokens: 500` yields 0 cached tokens and fails the check.
- A response whose only cache field is a cache-write field has proof but 0 cached tokens, so it fails after the first call.
- With strict mode off, no cache check is made and missing usage blocks are tolerated.
