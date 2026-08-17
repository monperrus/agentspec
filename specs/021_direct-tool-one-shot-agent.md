# One-shot agent from direct tool declarations

A library caller can run a one-shot task without writing an agent spec. They pass the task, the model connection (model name, endpoint, optional auth configuration) and a list of tool declarations. The agent builds the spec in memory, registers the tool implementations and runs the task exactly like a one-shot run of that spec.

## Behaviour
- Required inputs: the task text, the model name, the endpoint and a list of tool declarations (see declarative tool definitions). The auth configuration is optional.
- The in-memory spec has exactly these keys: `model`, `endpoint`, `inferred_tool_schema` (the `tools` array converted from the declarations), `tool_dispatch` (the matching dispatch table), and `auth` only when an auth configuration was given.
- Before the run, each declaration's implementation is registered in the implementation registry under its own name, overwriting any existing entry with that name, so that dispatch can resolve it.
- The task then runs through the normal one-shot path with that spec and task. The caller can pass every option of a one-shot run (non-interactive mode, session id, cache key, system prompt supplement, output-token limit, strict cache proof, event callback, tool executor, compaction settings, minimum cacheable tokens, durable journal, injected client). Each option is forwarded unchanged and has the same default as in a one-shot run.
- The result is the one-shot session result, unchanged.

## Edge cases
- When no auth configuration is given, the spec has no `auth` key at all. It does not get a null value. Key resolution then follows the normal rules.
- The built spec is validated like any other spec before any request is made.
- The implementations stay registered after the run finishes.
