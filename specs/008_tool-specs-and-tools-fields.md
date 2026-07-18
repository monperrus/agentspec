# Tool specs and tool list in the agent spec

An agent spec can declare its tools as two parallel lists: `tool_specs`, the OpenAI-compatible tool definitions offered to the model, and `tools`, the names of the built-in implementations that handle them, paired by index. The dispatch table, including argument-name translation, is derived from these two lists, so a spec no longer needs a hand-written `tool_dispatch`. The older `inferred_tool_schema` and `tool_dispatch` fields keep working.

## Behaviour
- Before a spec is validated, used to create a client, or used to start a session, it is normalized. Normalization never modifies the caller's spec; it works on a copy.
- Tool definitions are taken from `tool_specs`; if that field is absent, from `inferred_tool_schema`; if both are absent, the list is empty. After normalization both names refer to the same list, so every rule stated for `inferred_tool_schema` (validation, offered tools, aliases) applies to `tool_specs`.
- A `tool_specs` entry may be `{"type": "function", "function": {...}}` or the bare function object. Its tool name is `function.name`, and its parameter names are the keys of `function.parameters.properties`, in declaration order.
- When `tools` is present, it replaces any `tool_dispatch` in the spec. Entry *i* of `tools` gives the implementation for entry *i* of `tool_specs`, and the derived dispatch entry is keyed by that tool's name.
- The argument mapping is derived by position. The *k*-th parameter name of the tool spec maps to the *k*-th named parameter of the implementation, skipping receiver and variadic parameters. Only pairs whose names differ go into `param_map`; the other arguments pass through unchanged. For example, `str_replace(path, old_str, new_str)` paired with the string-replacement implementation yields `{"old_str": "old", "new_str": "new"}`.
- When `tools` is absent and `tool_dispatch` is absent or empty, each tool spec named `read_file`, `write_file`, `str_replace` or `execute_shell_command` gets the built-in default dispatch entry for that name. Other tool specs get no dispatch entry, so calling them is fatal.
- When `tools` is absent and `tool_dispatch` is non-empty, `tool_dispatch` is used as given.
- The default spec generated for a `run://` backend (held in memory, and written to disk only on a forced re-probe) has `tool_specs` (the four default tools) and `tools` (the built-in read, write, string-replace and shell implementation names, in that order), plus `status: "default"` and structured tool calls. It no longer contains `inferred_tool_schema` or `tool_dispatch`.
- A library caller can run one task given either a loaded agent spec, or a model and an endpoint (the spec is then loaded, optionally forcing a re-probe). Supplying both a spec and a model/endpoint is an error, as are supplying neither or omitting the task.

## Edge cases
- All of these fail with an invalid-spec error before the session starts:
  - `tools` is not a list of strings;
  - `tools` and `tool_specs` have different lengths (`'tool_specs' has <n> entries but 'tools' has <m>.`);
  - a `tools` entry names no known implementation (`Unknown tool function '<name>' in 'tools'.`);
  - a `tool_specs` entry paired with `tools` has no `function.name` (`Every 'tool_specs' entry must define function.name.`);
  - a tool spec declares more parameters than its implementation accepts.
- A tool spec with fewer parameters than its implementation is accepted. The extra implementation parameters are simply never mapped.
- `tools: []` with an empty `tool_specs` yields an empty dispatch table. The spec is then still rejected for having no tool schema.
