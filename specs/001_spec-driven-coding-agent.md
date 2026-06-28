# Spec-driven coding agent

The agent is a coding agent that works with any OpenAI-compatible chat-completions endpoint. It is driven by a JSON agent spec that describes, for one model, which tools to offer, how that model delivers tool calls, and which built-in tool implementation handles each tool. The agent runs one task non-interactively or as an interactive REPL, dispatches the model's tool calls to local implementations, logs each session, and can resume a session later.

## Behaviour

### Agent spec (JSON)
- Recognised top-level fields: `model`, `endpoint`, `status`, `inferred_tool_schema` (OpenAI `tools` array), `behaviour.call_delivery_mode`, `tool_dispatch`, `aliases`, `options`, `provider`, `provider_api_support.streaming.supported`, `max_output_tokens`, `max_rpm`, `max_input_token_price_per_million`, `max_output_token_price_per_million`, `disabled`, `comment`, `auth`, `key_env`, `keyring_service`, `keyring_username`.
- `tool_dispatch` maps a model-facing tool name to `{"python_function": <built-in implementation name>, "param_map": {<model arg>: <implementation arg>}}`. Arguments not listed in `param_map` pass through unchanged.
- A spec is rejected before running if `disabled` is true (error message is `comment`, default "This agent spec is disabled.") or if `inferred_tool_schema` is missing or empty ("No tool schema for '<model>' — probe likely failed.").

### Loading a spec
- If the model argument ends in `.json`, it is read as a spec file path; a relative path is resolved against the installation's project root.
- Otherwise the spec is looked up in the project root as `agent_spec_<safe>.json`, then `inferred_tool_schema_<safe>.json`, then `tool_schema_<safe>.json`, where `<safe>` is the model name with `/` and `:` replaced by `_`.
- The agent never probes a model itself. If no spec file exists (or a re-probe is forced, which skips the cached files), it fails with an invalid-spec error: "No agent spec found for '<model>'. Run `llmprobe <model>` (with --endpoint <endpoint> if needed) to probe the model and generate a spec file." No spec file is written.
- `run://<binary path>` as endpoint or model selects a local subprocess backend. If no cached spec exists, a default spec is generated and saved with `status: "default"`, structured tool calls, and four tools: `read_file(path, offset?: integer, limit?: integer)`, `write_file(path, content)`, `str_replace(path, old_str, new_str)`, `execute_shell_command(command)`. Only `path` is required for `read_file`; `offset` is described as the line number to start reading from (0-indexed, negative values count from the end of the file) and `limit` as the maximum number of lines to read. The default `tool_dispatch` for `str_replace` has `param_map` `{"old_str": "old", "new_str": "new"}`, so the model's `old_str`/`new_str` arguments reach the string-replacement implementation; the other three have an empty `param_map`.
- Default endpoint: `https://openrouter.ai/api/v1`.

### Aliases and options
- `aliases` maps an alias name to a canonical tool name. At session start the canonical dispatch entry is copied under the alias, and in structured mode a copy of the canonical tool schema is added under the alias name. An alias that already has its own `tool_dispatch` entry keeps it. An alias whose canonical tool has no dispatch entry is skipped with a warning.
- `options` is an array of strings. `exclude-prompt_cache_key` suppresses the `prompt_cache_key` request field; the `user` field is still sent.

### Session
- A session id is 12 random hex characters, or the id of the session being resumed.
- Call delivery mode is `structured_tool_calls` (default) or inline. In structured mode the system prompt is "You are a helpful coding agent. Use the provided tools to complete the task. When finished, reply in plain text." In inline mode the system prompt instructs the model to reply with ONLY a JSON object `{"name": ..., "arguments": {...}}` per call, lists one example per tool with `<arg>` placeholders, and asks for a plain-text summary when done.
- At session start, the system prompt is extended in this order, each part separated from the previous by a blank line: an optional caller-supplied supplement, then the contents of the user's global instructions file `~/.claude/CLAUDE.md` (in the home directory) if it exists, then the contents of `AGENTS.md` in the current working directory if it exists. A missing file is silently skipped.
- In non-interactive mode, tools that map to the ask-the-user implementation are removed from the offered tools; if the model still calls one, the result is `ERROR: user interaction is disabled (--non-interactive)`.
- Every request uses temperature 0 and sends a cache key as `user` and (unless excluded) `prompt_cache_key`. The cache key defaults to the session id; a caller may supply a stable one without resuming the conversation. `max_tokens` is set from `max_output_tokens` when configured. In structured mode `tools` and `tool_choice: "auto"` are sent.
- Against an OpenRouter endpoint, requests also send `usage: {"include": true}` and `provider` from the spec. If the spec has no `provider`, the session pins to the provider that served the first response (`{"order": [<provider>], "allow_fallbacks": false}`) and emits a `provider_pinned` event.
- When the spec says streaming is supported, content and reasoning deltas are emitted as they arrive.

### Turn loop
- A turn appends the user task and then repeatedly calls the model:
  - Structured mode with tool calls: the assistant message with its tool calls is recorded, each call is dispatched in order (unparseable arguments become `{}`), and each result is added as a `tool` message.
  - Inline mode: every JSON object in the reply that has `name` (or `function_name`) and a dict `arguments` (or `parameters`) is a call. Several calls in one reply are dispatched in order, and their results go back as one user message: `Tool results:\n` followed by `[<name>] <result>` blocks separated by blank lines.
  - Otherwise the reply is the final answer and the turn ends.
- The turn result has the session id, the final reply (the last non-empty assistant content, or none), cumulative usage (`prompt`, `completion`, `total`, `cached`, `cache_write`) and the message list.
- Token budget: each turn has a budget of 3,000,000 effective tokens. Each response adds `max(0, prompt_tokens - cached_tokens) + completion_tokens` to the count; cached prompt tokens do not consume budget. The cumulative usage `total` still records the raw API total. When the count exceeds the budget, a `token_limit` event fires with the message `[stopped after exceeding <limit> effective tokens (used <used> effective)]`; if the session has cached tokens, ` (<raw total> raw API total, <cached> cached)` is inserted after `effective` inside the parentheses. Numbers use thousands separators. Interactively, the user is asked whether to double the budget (`y`/`yes` continues). Otherwise the turn stops with no final reply.
- Built-in file tools (read, write, string replace, update file, list directory, search files given a plain directory path) expand a leading `~` or `~user` in the path argument to the home directory before accessing the filesystem. Result messages still quote the path as the model gave it.
- The built-in shell-command tool runs the command string with `/bin/bash`, not `/bin/sh`, so bash syntax (e.g. `[[ ... ]]`, brace expansion, `source`, process substitution) works.
- A call to a tool with no `tool_dispatch` entry is fatal: the error is logged and the process exits with status 2. An unknown implementation name or bad arguments return an `ERROR: ...` string to the model.
- If a model call fails (network, HTTP or API error), the turn does not crash. An `error` event fires with text `API error: <message>`, an `error` log record is written with field `error` holding the same text, and the turn ends at once with the result so far (no final reply unless one was already produced).
- In the REPL and in the interactive CLI loop, any other unexpected error during a turn fires an `error` event with the error message and writes an `error` log record. The loop then continues to the next prompt, and the conversation snapshot is still saved.
- Ctrl-C during a turn kills the running tool subprocess (its whole process group) and aborts the turn back to the prompt. At the prompt, Ctrl-C is ignored. A caller can also cancel a turn from another thread; the turn stops at the next model-call boundary.

### Events
- The agent reports progress as typed events, each carrying a pre-formatted display string: `tool_call`, `tool_result`, `usage`, `token_limit`, `final_answer`, `session_usage`, `session_resumed`, `provider_pinned`, `content_delta`, `reasoning_delta`, `content_stream_end`, `reasoning_stream_end`, `error` (displayed as `Error: <text>`). A caller can supply its own event handler. By default, the display string is printed (`token_limit` to stderr, and streaming deltas without a newline).
- A displayed tool result shows at most 20 lines, followed by a count of the remaining lines. Displayed tool-call arguments are cut at 200 characters.

### Logging and resume
- Each session appends JSON Lines records to `~/.local/share/agent_probe/<safe model>/<YYYY-MM-DD>/<HHMMSS>_<session id>.jsonl`. Record types: `session_start`, `session_resumed`, `user`, `usage`, `tool_call`, `tool_result`, `fatal_error`, `error`, `provider_pinned`, `assistant`, `session_end`. Every record has `ts` (ISO seconds) and `cwd`. A `tool_result` record has the tool `name`, the name of the implementation that handled it, and `result`, the full result text returned to the model.
- After a task, the conversation is saved to `~/.local/share/agent_probe/<safe model>/<session id>_messages.json`, but only if it has at least one non-system message. Messages that lack `ts` get one.
- Resuming by session id loads that snapshot. If none exists for the current model, the most recently modified snapshot with that id under another model is used, with a notice. If no snapshot is found anywhere, a warning is shown and the session starts fresh.
- A session can also be resumed by a short id of the form `<prefix>-<8 chars>` (any id whose last `-`-separated part is 8 characters long). When no snapshot file matches the id exactly, the short id matches a snapshot whose real session id hashes, as the first 8 lowercase hex digits of SHA-256 over its UTF-8 bytes, to that last part. The prefix is ignored. The current model's snapshots are searched first. Other models' snapshots are searched only when no other model has a snapshot with that exact id.
- The agent shows the user a resume command, i.e. the shell command that continues the current session. By default it is `<program> <model> --session <session id>`, where `<program>` is the name the CLI was invoked as (for the REPL entry point, `agent-probe`).
- An environment variable that overrides the resume command lets a wrapper script present itself: when it is set and non-empty, the resume command is `<value> --session <session id>`, with no model argument, because the wrapper is assumed to choose the model itself.

### Credentials and client
- `run://` specs use the subprocess backend. `auth: "opencode-github-copilot"` reads `github-copilot.access` from `~/.local/share/opencode/auth.json` and sends it in the `X-API-Key` header.
- HTTP clients enforce a client-side rate limit per endpoint base URL. All clients for the same base URL share one limit, and the first client created for that URL sets it. The limit is the spec's `max_rpm` (requests per minute) when present, and 40 otherwise. `run://` subprocess backends are not rate-limited.
- Before each HTTP request attempt (including retries after HTTP 429), if the limit's number of requests has already been made within the last 60 seconds, the agent prints `  [rate-limit] <N> RPM limit reached — waiting <seconds, 1 decimal>s …` and waits until the oldest of those requests is 60 seconds old. Waiting clients are served one at a time.
- Otherwise the API key comes from, in order: the keyring (`keyring_service`/`keyring_username`), then the environment variable named by `keyring_username` uppercased with `-` replaced by `_`, then the variable named by `key_env`, then `OPENROUTER_API_KEY`.

### Pricing check
- Before running, the current per-million-token input and output prices are fetched: from the OpenRouter models API for OpenRouter endpoints, or from the Azure Retail Prices API for Azure endpoints (the Azure data is cached on disk for 7 days, and global prices are preferred over data-zone prices). The prices are displayed.
- If a price exceeds `max_input_token_price_per_million` or `max_output_token_price_per_million`, the run is refused with a pricing-limit error that names the direction (input or output), the current price and the limit.
- `run://` and other endpoints show "no price check". If the price fetch fails, a warning is shown and the run proceeds.

## Edge cases
- A tool call whose arguments are not valid JSON is dispatched with empty arguments rather than failing.
- Inline JSON objects that fail to parse are skipped; scanning resumes at the next `{`.
- A resumed session keeps the resumed id as its session id; if no snapshot is found, it starts fresh under that id.
- A fresh session with only the system message leaves no snapshot.
- An exact snapshot id match always takes precedence over short-id resolution. An id with no `-`, or whose last part is not 8 characters long, is never treated as a short id.
- If the resume-command override variable is set but empty, the default resume command is used.
