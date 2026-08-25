# Bounded background shell tools

A ready-made pair of model-facing tools, `nohup` and `nohup_query`, runs shell commands in the background like the asynchronous shell execution implementations, but with a bounded run time. A caller adds both tools, with their model-facing definitions and their dispatch, to any agent spec in one idempotent step.

## Behaviour
- The implementation registry has an implementation named `t_nohup`, next to `t_execute_async` and `t_query_exec`.
- `t_nohup` takes `command` (string, required) and `timeout` (integer minutes, optional, default 10). It starts, with no delay, the shell command `timeout <timeout × 60> <command>` through the asynchronous start implementation, and returns exactly what that start returns (same JSON fields, inline output rule, FIFO stdin, completion event). E.g. `sleep 30` with the default becomes `timeout 600 sleep 30`; with `timeout` 3 it becomes `timeout 180 sleep 30`.
- The `timeout` value is truncated to an integer before converting to seconds.
- Tool `nohup` (dispatched to `t_nohup`):
  - Parameters: `command` (string, "Shell command to run.", required) and `timeout` (integer, "Maximum minutes the command may run before being killed (default <N>).", optional).
  - Description: `Start a shell command asynchronously, like nohup(1). Returns tool_exec_id and local file paths for stdin (FIFO), stdout, and stderr. Write to stdin_localfile to send input to the running process. Execution is bounded: the command is killed after `timeout` minutes (default <N>). If the command finishes within 100 ms and both outputs are under 4096 bytes, stdout/stderr are inlined immediately.`
- Tool `nohup_query` (dispatched to `t_query_exec`):
  - Parameter: `tool_exec_id` (string, "The tool_exec_id returned by nohup.", required).
  - Description: `Poll a command started with nohup. When completed, includes returncode and inlines stdout/stderr if both are under 4096 bytes; otherwise reports file sizes.`
- `<N>` in the descriptions is the default bound the caller chose when enabling the tools (10 unless overridden). The definitions are listed in the order `nohup`, `nohup_query`.
- Enabling the tools on an agent spec modifies the spec in place and returns it:
  - The tool definition list is the spec's existing `tool_specs`, else its `inferred_tool_schema`, else empty. The two definitions are appended, and the resulting list is stored under both `tool_specs` and `inferred_tool_schema`.
  - If the spec has a `tools` list (implementation names parallel to the definitions), `t_nohup` and `t_query_exec` are appended to it, in that order, so it stays aligned with the definitions.
  - Otherwise `tool_dispatch` is created if missing, and gets `nohup` → implementation `t_nohup` and `nohup_query` → implementation `t_query_exec`, each with an empty parameter map.

## Edge cases
- Enabling is idempotent: if a tool named `nohup` is already among the definitions, the spec is left unchanged (no duplicate definitions, `tools` or dispatch entries).
- The default bound passed when enabling only changes the descriptions shown to the model; a call that omits `timeout` is still bounded at 10 minutes.
- When the bound expires, the command is terminated by `timeout`, so the completion event and poll report its exit status (124 for a normal timeout kill).
- The low-level `t_execute_async` / `t_query_exec` implementations remain available under their names, unchanged and unbounded.
