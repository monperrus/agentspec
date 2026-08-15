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
  - Cached tokens ≤ 0 → the turn aborts with the error `Strict cache mode requires cached_tokens > 0 after the first LLM call, but the server reported no cache hit.`, unless the minimum-cacheable floor below applies.
- Minimum cacheable prompt size: providers cache nothing below a provider-specific prompt size (e.g. ~4096 tokens for small Claude models, ~1024 for some GPT models). The floor is set by the agent spec field `min_cacheable_tokens` (integer, prompt tokens), overridable per session by a programmatic argument (single task, REPLs, session creation) or by the CLI flag `--min-cacheable-tokens N`; an explicit argument or flag wins over the spec field. Default is 0 (no floor).
  - When the floor is nonzero and a response has cache proof, cached tokens ≤ 0, and its `usage.prompt_tokens` (0 if absent) is strictly below the floor, the turn does not abort: the agent emits a `cache_below_minimum` event with `prompt_tokens` and `min_cacheable_tokens`, displayed dim as `No cache hit, but prompt (<prompt_tokens> tokens) is below the provider's minimum cacheable prefix (<min_cacheable_tokens> tokens); not a caching failure.`, and the response is processed normally.
- In strict mode, a response with no usage block after at least one counted call aborts the turn with the error `Strict cache mode requires usage metadata on every LLM call after the first, but the server returned no usage block.`
- Cold resume: before either check above, the agent measures the age of the session's most recent message that carries a timestamp (`ts`, ISO 8601 local time), as now minus that timestamp. If the age is strictly greater than 3600 seconds, the prefix cache is assumed to have expired:
  - A response with a usage block that lacks cache proof or reports cached tokens ≤ 0 does not abort; the agent emits a `cache_cold` event with `age` (whole seconds) and the dim display line `Prefix cache expired (last message <age>s old); this turn was not served from cache and will re-process the prompt.` and the response is processed normally.
  - A response with no usage block does not abort; the agent emits a `cache_cold` event with the dim display line `Prefix cache expired (last message <age>s old); usage metadata unavailable this turn.` and processing continues.
  - A response with cache proof and cached tokens > 0 passes silently; no `cache_cold` event is emitted.
- An abort emits an `error` event (displayed as `Error: <text>`), appends an `error` log record, and ends the turn normally with the turn result (session id, the last non-empty assistant reply so far, cumulative usage, messages). It does not crash the REPL or the process.

## Edge cases
- The first counted call is never checked, so a cold cache on the first call is allowed.
- A response without a usage block is not counted; if it is the first response of the session, it passes.
- The failing response is discarded: its usage is not added to totals or emitted, and its tool calls or reply are not processed.
- A present-but-zero field wins over later candidates: `details.cached_tokens: 0` with `usage.cache_read_input_tokens: 500` yields 0 cached tokens and fails the check.
- A response whose only cache field is a cache-write field has proof but 0 cached tokens, so it fails after the first call.
- With strict mode off, no cache check is made and missing usage blocks are tolerated; no `cache_cold` event is emitted.
- The first counted call never emits `cache_cold`, since it is never checked.
- An age of exactly 3600 seconds or less is warm: misses abort as usual.
- If no message has a timestamp, or the most recent timestamp cannot be parsed, the age is unknown and the strict checks apply unchanged.
- A prompt size equal to or above the floor with no cache hit aborts as usual; a floor of 0 or null disables the relaxation.
- The floor does not excuse a missing cache-proof field or a missing usage block; those still abort.
- The cold-resume check runs first: if the session is cold, `cache_cold` is emitted instead of `cache_below_minimum`.
- A cache hit (cached tokens > 0) never emits `cache_below_minimum`, whatever the prompt size.
- The cold-resume relaxation is not limited to one event: every qualifying check while the age exceeds the threshold emits its own `cache_cold` event.
