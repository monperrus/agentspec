# Slash commands

In the interactive REPL and the interactive CLI loop, a line that starts with `/` is a slash command. The agent handles it locally and never sends it to the model. The built-in commands are `/c`, `/clear`, `/compact`, `/model`, `/usage`, `/hooks` and `/help`. A caller can register more commands or remove existing ones. Each command has a name and a one-line description.

## Behaviour
- Before a non-empty input line is treated as a task, the agent checks the exit words (`exit`, `quit`, `q`) first, then slash commands.
- A line is a slash command if it starts with `/` once surrounding whitespace is removed. The command name is the first word after the `/`, matched case-insensitively. Everything after the first run of whitespace is the argument string, which may be empty.
- An unknown name prints `Unknown command: /<name>. Type /help for available commands.` The line is still consumed and does not go to the model.
- If a command fails, the agent prints `Error running /<name>: <message>` and the loop continues.
- After a slash command runs, the conversation snapshot is saved under the same rules as after a turn.
- `/help` prints `Available slash commands:` followed by one line per registered command, sorted by name: `  /<name padded to 12>  <description>`. The built-in descriptions are:
  - `c`: "Retry an interrupted turn without adding a user message."
  - `clear`: "Reset the session message history (keep system prompt)."
  - `compact`: "Summarize older history into a compact continuation summary."
  - `model`: "List available models or switch: /model <model-id>."
  - `usage`: "Show token usage for the current session."
  - `hooks`: "List configured lifecycle hooks and their sources."
  - `help`: "Show this help message."
- `/c` (continue) prints nothing. Right after it, the REPL runs a new turn on the conversation as it stands, without adding a user message. The request re-sends the existing transcript unchanged. No user message is merged or appended, and no `user` log record is written. The session journal's `turn_start` record has a `null` task. This turn replaces the usual snapshot save after a slash command. The turn itself saves the snapshot as any turn does, and it gets the same interruption, error and event handling.
  - Example: the user sends `work`, the turn is interrupted, then the user enters `/c`. The retry request's only user message is `work`.
  - In the asynchronous REPL (see type-ahead input queue), a `/c` queued during a turn adds a retry to the pending instructions, where a text line would go. The retry runs as a follow-on turn in the same way.
  - Any front end that embeds slash-command dispatch (not only the built-in REPL) can hand the dispatcher a retry action. When a command asks for a retry, the dispatcher clears the request and invokes that action itself, once, before reporting the line as handled. The front end therefore needs no knowledge of how the request is signalled, and `/c` retries the turn in every front end that supplies the action.
  - Without a retry action, the dispatcher leaves the retry request pending on the session so the caller can resolve it (backward-compatible behaviour).
  - When the retry action runs, the usual post-command snapshot save is skipped; the retried turn saves it instead.
- `/compact` compacts the history right away, using manual compaction (see context compaction) with the session's current model. The argument string is ignored. If the history was compacted, it prints `Context compacted: <before> → <after> messages.`, where the counts are total messages including system messages. Otherwise it prints `Nothing to compact.`
- `/clear` removes every message except the system messages. If there are none, it keeps only the first message. It also sets the cumulative usage totals (`prompt`, `completion`, `total`, `cached`, `cache_write`) to 0, resets the compaction watermark (see context compaction) so a new growth cycle can trigger compaction, fires `SessionEnd` hooks with reason `clear` (see lifecycle hooks), and prints `Context cleared. Session history has been reset.`
- `/hooks` lists the session's configured lifecycle hooks. With none, it prints a one-line `No hooks configured.` notice naming the places hooks can come from. Otherwise:
  - if hooks are disabled for the session, it first prints a yellow line `hooks are disabled for this session (hooks_enabled=false)`;
  - it prints one bold heading per event, sorted by name, with a dim ` (never fires: no such lifecycle point yet)` after events that never fire;
  - under each heading, one line per entry in configuration order: two spaces, the matcher (`*` when empty) padded to 24, a space, the handler (the command plus any exec-form args, or `python:<function name>` for an in-process function), two spaces, and the dim source in brackets (the file path, `inline`, or `api`);
  - when context is still queued, a final dim line `pending context: <n>, queued async results: <m>`.
- `/model <id>` sets the session model to `<id>`, taken verbatim with surrounding whitespace trimmed. It prints `Model switched from <old> → <new>` and `The next turn will use the new model.` Every later turn in that session calls the new model.
- `/model` with no argument lists the models the endpoint offers:
  - The endpoint is the spec's `endpoint`, or the client's base URL if the spec has none.
  - The URL is derived from the endpoint: trailing `/` is stripped, a trailing `/v1` is removed, and `/models` is appended unless the URL already ends with `/models`. The request is an HTTP `GET` with a 15-second timeout. It sends `Authorization: Bearer <api key>` when a key is known, plus a `User-Agent` header that identifies the agent.
  - The response is either a JSON list of model objects or an object whose `data` field holds that list.
  - The output is `Available models (<n>):`, then one line per model showing its `id`, else its `name`, else the whole object. The current model is marked with `*`. The last line is `To switch: /model <model-id>`.
- `/usage` prints `Session token usage:` followed by:
  - the session's trajectory identifier, as `  trajectory: <id>` (`unknown` if the session has none);
  - the prompt token count;
  - the cached token count and its percentage of prompt tokens (only if non-zero);
  - the cache-write token count (only if non-zero);
  - the completion token count;
  - the total token count;
  - the number of non-system messages, as `messages: <n> (excl. system)`.
  - Numbers use thousands separators.

## Model-callable slash command tool
- The built-in commands can also be offered to the model as one ready-made tool named `slash_command`. An agent opts in by adding it to its tool list, and it is dispatched like any other tool.
- Tool description given to the model: `Run a slash command. command: one of clear, compact, model, usage, hooks, help. For 'model', pass a model-id in args to switch; omit args to list.`
- Parameters: `command` (required string, enum `clear`, `compact`, `model`, `usage`, `hooks`, `help`, description `Slash command to run.`) and `args` (optional string, default empty, description `Optional argument (e.g. model-id for 'model').`).
- Before the tool is used, the agent binds it to the live session, the model client and the current model name. It usually does this right after creating the session.
- A call runs the same built-in behaviour as the matching REPL command, with `args` as the argument string, on the bound session. For example, `clear` resets the bound session's history and usage totals, and `model` with an id switches the bound session's model.
- The tool result is the text the command would have printed, with leading and trailing whitespace removed. It is not printed to the console. The structured result carries the same text in a `result` field.

## Edge cases
- If no endpoint can be determined, `/model` prints `Cannot determine endpoint URL to query /models.`
- For a `run://` subprocess backend, `/model` with no argument prints `/models is not available for subprocess backends.` and sends no request.
- If fetching the model list fails, `/model` prints `Failed to fetch models from endpoint: <message>` and then `The endpoint may not support GET /models.` If the list is empty, it prints `No models returned by the endpoint.`
- `/model <id>` does not check whether the model exists and does not reload the agent spec. The session keeps its tool schema and dispatch table.
- The cached percentage is 0 when there are no prompt tokens.
- If the tool gets a `command` outside the six built-ins, it runs nothing and returns `ERROR: unknown command '<command>'. Valid: clear, compact, model, usage, hooks, help`. Commands registered by a caller are not reachable through the tool.
- `/c` ignores its argument string. It does not check whether the last turn was actually interrupted: it always re-sends the current transcript as is, even when that transcript ends with an assistant message.
- The retry action is never invoked for commands other than `/c`, nor when a command fails with an error.
- `/c` is not one of the commands the `slash_command` tool offers to the model.
- A line such as `/tmp/file` counts as a slash command (the unknown command `tmp/file`), so it never reaches the model.
- A bare `/`, or `/` followed only by whitespace, has no command name. It prints `Empty command. Type /help for available commands.`, is consumed like any slash command, does not go to the model, and the loop keeps running. It must never crash the turn.
