# Declarative tool definitions

A library caller can declare a custom tool once (model-facing name, description, implementation, optional parameter schema, optional argument-name mapping) and obtain from a list of such declarations both the OpenAI-compatible `tools` array and the matching `tool_dispatch` table. These can replace the built-in tool set by being placed in an agent spec as `tool_specs` (or its legacy name `inferred_tool_schema`) and `tool_dispatch`.

## Behaviour
- A tool declaration has: a model-facing name, a description, an implementation, an optional parameter schema (a JSON Schema object) and an optional argument-name mapping (model argument name → implementation argument name).
- An implementation returns a text result and a metadata object containing at least `result`.
- Converting a list of declarations yields, in declaration order, one schema entry per tool of the form `{"type": "function", "function": {"name": <name>, "description": <description>, "parameters": <parameter schema>}}`, and one dispatch entry per tool keyed by its name: `{"python_function": <implementation name>, "param_map": <mapping>}`.
- The schema always uses the model-facing argument names; the mapping only affects dispatch.
- When no parameter schema is given, it is inferred from the implementation's declared parameters as `{"type": "object", "properties": {...}, "required": [...]}`:
  - Each named parameter becomes a property. Receiver parameters (the instance or class) and variadic positional/keyword parameters are skipped.
  - Declared types map as string → `"string"`, integer → `"integer"`, floating point → `"number"`, boolean → `"boolean"`; any other or missing type → `"string"`.
  - Parameters without a default value are listed in `required`, in declaration order.
- When no mapping is given, `param_map` is the identity mapping over the properties of the (given or inferred) parameter schema.
- The implementation name written to `python_function` is the implementation's own name. A caller can register a list of declarations in the implementation registry under those names, so that dispatch can resolve them; registering overwrites any existing entry with the same name.
- A tool can also be described by a tool-spec block embedded in its implementation's documentation text. The block starts at the first line whose trimmed content is exactly `Tool spec:` and consists of the following indented lines, for example:
  ```
  Tool spec:
      name: read_file
      description: Read and return the contents of a file.
      parameters:
          path:
              type: string
              description: Path to the file.
              required: true
  ```
  Parsing a block yields `name`, `description`, `parameters` (parameter name → an object with the `type` and/or `description` given under it), with parameters in block order, and `required` (the names of parameters marked `required: true`, in declaration order, without duplicates). Values are the text after `key:`, trimmed. `required` is true when its value is `true`, `yes` or `1` (case-insensitive); any other value leaves the parameter optional. Unknown keys are ignored.
- A caller can convert a parsed tool-spec block into one OpenAI-compatible schema entry: `{"type": "function", "function": {"name": <name>, "description": <description>, "parameters": {"type": "object", "properties": <parameters>, "required": <required>}}}`. The `required` marker never appears inside a property; properties keep only their `type` and `description`.
- A caller can scan a collection of implementations and obtain, for every implementation whose documentation contains a tool-spec block, a mapping from the implementation name to its parsed block. Implementations without a block are omitted. Combined with the conversion above, this lets a host publish a schema for a set of documented tools without re-describing their parameters.
- The built-in read, write, string-replace and shell implementations carry tool-spec blocks whose conversion reproduces their entries in the default tool set: same parameter names, types and `required` lists (`read_file`: `path`; `write_file`: `path`, `content`; `str_replace`: `path`, `old_str`, `new_str`; `exec_shell`: `command`). For `read_file` the conversion is identical to the default entry. Descriptions may differ in wording but are never empty.
- The extra library tools also carry blocks:
  - `glob`: returns the paths matching a glob pattern, one per line; parameter `pattern` (string, required).
  - `list_dir`: lists a directory's entries, one per line, prefixed by `d` (directory) or `f` (file); parameter `path` (string, required).
  - `search_files`: searches file contents for a regular expression and returns the matching lines as JSON; parameters `pattern` (string, required) and `path` (string, optional, default `.`).
- Every parameter an implementation needs without a default is marked required in its block.
- A caller can obtain the built-in tool set (`read_file`, `write_file`, `str_replace`, `exec_shell`) as the same schema-array / dispatch-table pair, identical in shape to the result of converting declarations. Its implementations are already registered, so the pair can be passed to dispatch as is (e.g. writing then reading a file through it works) — useful for a host that only embeds the tool runtime, such as a server re-publishing the tools under another protocol.
- Each request for the built-in pair returns fresh, fully independent copies of both halves: a caller may mutate or extend them (clear the schema array, edit a `param_map`, add tools) without affecting later requests or the agent's own default tool set.
- The implementation registry can hold several implementations of the same capability. A caller can obtain two read-only maps that say which one to use:
  - The canonical map: model-facing tool name → the implementation name to build a tool set from. It has exactly seven entries: `read_file`, `write_file`, `str_replace`, `exec_shell`, `list_dir`, `glob` and `search_files` (content search), each mapped to the implementation that carries that tool's tool-spec block.
  - The superseded map: older implementation name → the canonical implementation name that replaces it. The older update-file implementation is replaced by the `str_replace` implementation; the older list-directory implementation by the `list_dir` one; and both the older search-files implementation (which, despite its name, looks up paths rather than file contents) and the find-files implementation by the `glob` one.
- Invariants: every canonical implementation is registered; scanning the registry's implementations for tool-spec blocks yields exactly the canonical map (block `name` → implementation name, no more, no less); every superseded implementation is still registered and callable (so specs and restored sessions naming it keep working), and every replacement it points to is a canonical implementation. Each superseded implementation's documentation names its replacement.
- The ask-the-user and background-shell tools are opt-in, described separately, and appear in neither map.
- A `tool_dispatch` entry's `python_function` may be either an implementation name, looked up in the implementation registry, or (for library callers building the table in memory) the implementation itself, which is called directly. `param_map` translation applies identically in both cases.

## Edge cases
- In the built-in pair, the schema entries list exactly the four built-in tool names (first entry `read_file`), each with `"type": "function"` and an object-typed `parameters`; `read_file`'s `param_map` is empty.
- An empty list of declarations yields an empty `tools` array and an empty `tool_dispatch` table.
- If the implementation's parameters cannot be introspected, the inferred schema is `{"type": "object", "properties": {}, "required": []}`.
- A dispatch entry with no `python_function`, or with a name absent from the registry, returns `ERROR: python_function '<value>' not found in TOOL_LIBRARY` to the model (not fatal); a tool with no dispatch entry at all is not fatal either and returns the unknown-tool error listing the available tools.
- Documentation that is empty, missing, or has no `Tool spec:` line yields no tool spec.
- In a tool-spec block, a missing `name` or `description` yields an empty string, and a missing `parameters` section yields no parameters, and a block with no `required: true` marker yields an empty `required` list (converted to `"required": []`). A parameter line with no sub-keys yields an empty object for that parameter.
- The block ends at the first non-blank line indented less than the first non-blank line after `Tool spec:`; blank lines inside the block are ignored.
- Two declarations with the same model-facing name: the later one's dispatch entry wins, while both schema entries are emitted.
