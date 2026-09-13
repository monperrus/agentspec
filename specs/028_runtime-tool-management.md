# Runtime tool management

The `/tool` slash command lets the user (or the model, through the `slash_command` tool) inspect and change the session's offered toolset while the session runs. A tool can be removed for the rest of the session, a removed tool can be restored exactly as it was, and any implementation in the implementation registry can be brought in as a new tool. Changes take effect from the next model request.

## Behaviour
- `/help` lists the command with the description `List / activate / remove tools at runtime: /tool list | /tool activate <name> | /tool remove <name>.`
- The first word of the argument string is the sub-command, matched case-insensitively. The rest, with surrounding whitespace trimmed, is the tool name. With no argument, the sub-command is `list`.
- `/tool`, `/tool list` and `/tool ls` print three groups:
  - `Active tools (<n>):`, then one line per offered tool, in offered order: `  ✓ <model-facing name>`, followed by two spaces and the first line of its description truncated to 70 characters, when it has a description.
  - Only if any exist, `Inactive (removed, re-activatable):`, then one line per tool removed earlier in the session, sorted by name: `  ✗ <name>   /tool activate <name>`.
  - Only if any exist, `Available in TOOL_LIBRARY (not active):`, then one line per registry implementation whose model-facing name is neither active nor inactive, sorted by name and without duplicates: `  - <name>   /tool activate <name>`. A registry implementation's model-facing name is the name declared in its tool-spec documentation if it has one, otherwise its implementation name with a leading `t_` removed.
  - Finally a blank line and `Usage: /tool list | /tool activate <name> | /tool remove <name>`.
- For both function tools and custom tools (see custom tool declarations), the model-facing name and description are read from wherever that tool shape keeps them.
- `/tool remove <name>` removes the active tool with that model-facing name from the offered tools and keeps its definition aside as inactive. It also removes the tool's dispatch entry, and the dispatch entries of every built-in legacy alias of it (e.g. `execute_shell_command` for `exec_shell`), and keeps them aside too. The model therefore cannot call a tool it can no longer see. It prints `Tool removed: <name>` and `Re-activate later with /tool activate <name>.`
- `/tool activate <name>` first maps a built-in legacy alias to its canonical name (e.g. `execute_shell_command` → `exec_shell`). Then:
  - if the tool is already active, nothing is added (idempotent);
  - else if it is inactive, its kept definition is appended to the offered tools unchanged, and its kept dispatch entries (its own and its aliases') are restored unchanged;
  - else if a registry implementation declares a tool spec with that model-facing name, a function tool is built from it: its declared description (the name if none), and one property per declared parameter with its declared type (default `string`) and description, every parameter required. A dispatch entry is installed that calls that implementation with arguments passed through unchanged;
  - else if `<name>` is an implementation name in the registry, a function tool is built from the implementation's signature (see declarative tool definitions). Its model-facing name is the implementation name with a leading `t_` removed, and its description is the first non-empty line of the implementation's documentation, or the model-facing name if there is none. Its dispatch entry is installed.
  - The new or restored tool is appended at the end of the offered tools. The name is no longer listed as inactive. The command prints `Tool activated: <name>` and `The next turn will offer it to the model.`
- Each successful activate writes a session log record `{"type": "tool_activated", "tool": <name>}`, and each successful remove writes `{"type": "tool_removed", "tool": <name>}`.
- In a non-interactive session, the ask-the-user implementations are never listed as available candidates.
- The model can run the same sub-commands through the `slash_command` tool with `command` `tool` and `args` `list`, `activate <tool_name>` or `remove <tool_name>` (see slash commands).

## Edge cases
- `activate` or `remove` with no tool name prints `Usage: /tool <sub-command> <tool_name>` and changes nothing.
- `/tool remove <name>` on a tool that is not active prints `Tool '<name>' is not active. Active: <comma-separated active names>` (`(none)` if empty) and changes nothing.
- `/tool activate <name>` for a name that matches nothing prints `Unknown tool '<name>'. Known: <sorted comma-separated names>`, where the names are the registry implementation names, the tool-spec model-facing names, and the active and inactive tool names (`(none)` if empty). Nothing changes.
- An unknown sub-command prints `Unknown /tool sub-command '<sub>'. Usage: /tool list | /tool activate <name> | /tool remove <name>`.
- Activating an already active tool still prints the activation message and writes the log record.
- Remove followed by activate restores the exact same definition and dispatch entries (including parameter-name mappings), but the tool moves to the end of the offered order.
