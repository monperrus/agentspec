# Custom tool calls

Besides standard function tool calls, the agent accepts custom tool calls (OpenAI custom tools, e.g. grammar-constrained output), whose payload is raw text rather than JSON arguments. Such calls are dispatched without JSON decoding, kept in their original shape in the conversation history, and rendered as readable text when a session is flattened for resume or replayed. Function calls behave exactly as before.

## Behaviour
- A tool call in a chat-completions response may have the shape `{"id": ..., "type": "custom", "custom": {"name": <tool name>, "input": <raw text>}}`. A tool call is treated as custom whenever it carries a non-empty `custom` object; otherwise it is a function call read from `function.name` and `function.arguments`.
- A single response may mix function calls and custom calls; they are dispatched in response order.
- In streamed responses, a tool-call delta carrying a `custom` object marks that call as custom; its `custom.name` and `custom.input` fragments are concatenated across deltas, in arrival order.
- A custom call is dispatched to the tool named `custom.name` with exactly one argument, `input`, whose value is the raw input text, unchanged. No JSON parsing is attempted, so input that is not valid JSON is passed through as is. The usual dispatch rules (`tool_dispatch`, `param_map`, missing-implementation errors) then apply as for function calls.
- The tool result is appended as a normal `tool` message referencing the call's `id`.
- In the assistant history message, a custom call is recorded as `{"id": <id>, "type": "custom", "custom": {"name": <name>, "input": <raw text>}}`, and a function call as `{"id": <id>, "type": "function", "function": {"name": <name>, "arguments": <arguments>}}`. Custom calls are therefore sent back to the provider in their original shape on later turns.
- When a snapshot is flattened to plain text for resume, a custom call becomes an assistant message with content `[Tool call: <name>(<input>)]`, with the raw input text in place of the arguments string.
- When a resumed session's history is replayed on screen, a custom call is displayed like a function call whose arguments are `{"input": <raw text>}`.

## Edge cases
- A custom `input` that is not a string (e.g. a JSON object) is serialized to JSON text (non-ASCII characters kept as is) and that text is used as the raw input.
- A missing custom `input` is the empty string; a missing custom `name` is the empty string when parsing a response, and `?` when flattening or replaying history.
- When flattening for resume, a function call with a missing `name` is shown as `?` and a missing `arguments` as the empty string.
