# Slash commands

In the interactive REPL and the interactive CLI loop, a line that starts with `/` is a slash command. The agent handles it locally and never sends it to the model. The built-in commands are `/clear`, `/model`, `/usage` and `/help`. A caller can register more commands or remove existing ones. Each command has a name and a one-line description.

## Behaviour
- Before a non-empty input line is treated as a task, the agent checks the exit words (`exit`, `quit`, `q`) first, then slash commands.
- A line is a slash command if it starts with `/` once surrounding whitespace is removed. The command name is the first word after the `/`, matched case-insensitively. Everything after the first run of whitespace is the argument string, which may be empty.
- An unknown name prints `Unknown command: /<name>. Type /help for available commands.` The line is still consumed and does not go to the model.
- If a command fails, the agent prints `Error running /<name>: <message>` and the loop continues.
- After a slash command runs, the conversation snapshot is saved under the same rules as after a turn.
- `/help` prints `Available slash commands:` followed by one line per registered command, sorted by name: `  /<name padded to 12>  <description>`. The built-in descriptions are:
  - `clear`: "Reset the session message history (keep system prompt)."
  - `model`: "List available models or switch: /model <model-id>."
  - `usage`: "Show token usage for the current session."
  - `help`: "Show this help message."
- `/clear` removes every message except the system messages. If there are none, it keeps only the first message. It also sets the cumulative usage totals (`prompt`, `completion`, `total`, `cached`, `cache_write`) to 0 and prints `Context cleared. Session history has been reset.`
- `/model <id>` sets the session model to `<id>`, taken verbatim with surrounding whitespace trimmed. It prints `Model switched from <old> → <new>` and `The next turn will use the new model.` Every later turn in that session calls the new model.
- `/model` with no argument lists the models the endpoint offers:
  - The endpoint is the spec's `endpoint`, or the client's base URL if the spec has none.
  - The URL is derived from the endpoint: trailing `/` is stripped, a trailing `/v1` is removed, and `/models` is appended unless the URL already ends with `/models`. The request is an HTTP `GET` with a 15-second timeout. It sends `Authorization: Bearer <api key>` when a key is known, plus a `User-Agent` header that identifies the agent.
  - The response is either a JSON list of model objects or an object whose `data` field holds that list.
  - The output is `Available models (<n>):`, then one line per model showing its `id`, else its `name`, else the whole object. The current model is marked with `*`. The last line is `To switch: /model <model-id>`.
- `/usage` prints `Session token usage:` followed by:
  - the prompt token count;
  - the cached token count and its percentage of prompt tokens (only if non-zero);
  - the cache-write token count (only if non-zero);
  - the completion token count;
  - the total token count;
  - the number of non-system messages, as `messages: <n> (excl. system)`.
  - Numbers use thousands separators.

## Edge cases
- If no endpoint can be determined, `/model` prints `Cannot determine endpoint URL to query /models.`
- For a `run://` subprocess backend, `/model` with no argument prints `/models is not available for subprocess backends.` and sends no request.
- If fetching the model list fails, `/model` prints `Failed to fetch models from endpoint: <message>` and then `The endpoint may not support GET /models.` If the list is empty, it prints `No models returned by the endpoint.`
- `/model <id>` does not check whether the model exists and does not reload the agent spec. The session keeps its tool schema and dispatch table.
- The cached percentage is 0 when there are no prompt tokens.
- A line such as `/tmp/file` counts as a slash command (the unknown command `tmp/file`), so it never reaches the model.
