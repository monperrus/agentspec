# Structured error diagnostics

Every error the agent reports, as an `error` event and as a log record, carries a fixed set of machine-readable diagnostic fields, so that consumers can classify failures (by error type, HTTP status, latency and backend) without parsing the human-readable message.

## Behaviour
- The diagnostic fields are:
  - `error_class`: the short name of the error's type as raised (e.g. `HTTPError`, `RuntimeError`, `RateLimitError`).
  - `http_status`: the integer HTTP status code of the failure, or null. It is taken from the error's own status code if it has one, else from the status code of the HTTP response attached to the error, else from the error's plain `code` value (as exposed by standard-library HTTP errors).
  - `error_code`: the provider's business error code as text (e.g. `"1305"` from a Z.ai rate-limit body), or null when the error carries none.
  - `error_message`: the provider's business error message as text, or null when the error carries none.
  - `elapsed_s`: seconds between the start of the failing model request and the moment the error is reported, rounded to 3 decimals, as a number; null when the error did not come from a timed model request.
  - `adapter`: `subprocess` when the session uses the subprocess backend or its endpoint starts with `run://`; otherwise `http`.
- The `error` event data contains `text`, the six diagnostic fields and `fmt`.
- The log record written for the same error contains `type`, `error` (the same text as the event's `text`), the six diagnostic fields with the same values as the event, `ts`, and `error_kind` when one applies (e.g. `rate_limit`).
- The fields are present on every error path:
  - A failed model call during a turn, including a rate-limit failure (`error` record; `elapsed_s` measured from the start of that call).
  - A failed or rate-limited compaction summary call (`compaction_error` record; `elapsed_s` measured from the start of the summary call).
  - A strict-cache abort on the first model call of a session (`error` record; `elapsed_s` null, `http_status` null).
  - An unexpected error during a REPL or interactive CLI turn (`error` record; `elapsed_s` null).

## Edge cases
- A status value that is missing or not an integer yields `http_status` null; the fields are still present.
- `elapsed_s` is never negative.
- A `run://` endpoint reports `adapter` `subprocess` even if the injected client is not the built-in subprocess backend.
