# Custom tool declarations

A library caller can declare a tool as an OpenAI custom (freeform) tool instead of a JSON-Schema function tool, by giving the declaration a format object (typically a grammar for endpoint-side constrained decoding). The tool's argument is then raw text. Custom entries also flow unchanged through the agent spec's `tool_specs`, aliases and inline mode.

## Behaviour
- A tool declaration may carry an optional custom format object. When it is absent, the declaration behaves exactly as a function tool (see declarative tool definitions).
- When a custom format is given, the schema entry is `{"type": "custom", "name": <name>, "description": <description>, "format": <format>}`. It has no `function` and no `parameters` key, and no JSON Schema is inferred from the implementation.
- The format object is re-emitted exactly as given. Both the flat form (e.g. `{"type": "grammar", "syntax": "lark", "definition": "..."}`) and the chat-completions nesting (`{"type": "grammar", "grammar": {"syntax": ..., "definition": ...}}`) are accepted and reach the endpoint unchanged.
- The implementation of a custom tool takes a single text argument named `input`. Its dispatch entry is `{"python_function": <implementation name>, "param_map": {"input": "input"}}`, so a custom tool call (whose dispatched arguments are `{"input": <raw text>}`) delivers the raw text directly.
- `tool_specs` entries of type `"custom"` are forwarded to the endpoint verbatim. For such an entry:
  - the tool name is read from its top-level `name`;
  - it declares no JSON-Schema parameters; when `param_map` is derived for a `tools` pairing, it is `{"input": "input"}`;
  - its argument properties are treated as `{"input": {"type": "string"}}`.
- In inline call-delivery mode, the system prompt's example for a custom tool is `{"name": <name>, "arguments": {"input": "<text>"}}`.
- An alias of a custom tool clones the custom schema entry with its top-level `name` replaced by the alias name (function tools keep having the name replaced under `function`).

## Edge cases
- An explicit argument-name mapping on a custom declaration takes precedence over the `{"input": "input"}` default.
- A custom `tool_specs` entry with no `name` has the empty string as its name; in the inline prompt example it is shown as `?`.
- A minimal format such as `{"type": "grammar"}` is forwarded as is; no validation of the format is performed.
