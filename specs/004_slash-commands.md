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
- Tool description given to the model: `Run a slash command. command: one of clear, model, usage, help. For 'model', pass a model-id in args to switch; omit args to list.`
- Parameters: `command` (required string, enum `clear`, `model`, `usage`, `help`, description `Slash command to run.`) and `args` (optional string, default empty, description `Optional argument (e.g. model-id for 'model').`).
- Before the tool is used, the agent binds it to the live session, the model client and the current model name. It usually does this right after creating the session.
- A call runs the same built-in behaviour as the matching REPL command, with `args` as the argument string, on the bound session. For example, `clear` resets the bound session's history and usage totals, and `model` with an id switches the bound session's model.
- The tool result is the text the command would have printed, with leading and trailing whitespace removed. It is not printed to the console. The structured result carries the same text in a `result` field.

## Edge cases
- If no endpoint can be determined, `/model` prints `Cannot determine endpoint URL to query /models.`
- For a `run://` subprocess backend, `/model` with no argument prints `/models is not available for subprocess backends.` and sends no request.
- If fetching the model list fails, `/model` prints `Failed to fetch models from endpoint: <message>` and then `The endpoint may not support GET /models.` If the list is empty, it prints `No models returned by the endpoint.`
- `/model <id>` does not check whether the model exists and does not reload the agent spec. The session keeps its tool schema and dispatch table.
- The cached percentage is 0 when there are no prompt tokens.
- If the tool gets a `command` outside the four built-ins, it runs nothing and returns `ERROR: unknown command '<command>'. Valid: clear, model, usage, help`. Commands registered by a caller are not reachable through the tool.
- A line such as `/tmp/file` counts as a slash command (the unknown command `tmp/file`), so it never reaches the model.
