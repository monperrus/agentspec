# Declarative tool definitions

A library caller can declare a custom tool once (model-facing name, description, implementation, optional parameter schema, optional argument-name mapping) and obtain from a list of such declarations both the OpenAI-compatible `tools` array and the matching `tool_dispatch` table. These can replace the built-in tool set by being placed in an agent spec as `inferred_tool_schema` and `tool_dispatch`.

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
- A `tool_dispatch` entry's `python_function` may be either an implementation name, looked up in the implementation registry, or (for library callers building the table in memory) the implementation itself, which is called directly. `param_map` translation applies identically in both cases.

## Edge cases
- An empty list of declarations yields an empty `tools` array and an empty `tool_dispatch` table.
- If the implementation's parameters cannot be introspected, the inferred schema is `{"type": "object", "properties": {}, "required": []}`.
- A dispatch entry with no `python_function`, or with a name absent from the registry, returns `ERROR: python_function '<value>' not found in TOOL_LIBRARY` to the model (not fatal); a tool with no dispatch entry at all remains fatal.
- Two declarations with the same model-facing name: the later one's dispatch entry wins, while both schema entries are emitted.
