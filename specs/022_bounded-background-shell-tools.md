# Bounded background shell tools

A ready-made trio of model-facing tools runs shell commands in the background like the asynchronous shell execution implementations, but with a bounded run time: `nohup` starts a command, `nohup_query` polls one, and `wait_for` blocks for a given duration instead of busy-polling, then reports every execution that finished meanwhile. A caller adds all three tools, with their model-facing definitions and their dispatch, to any agent spec in one idempotent step.

## Behaviour
- The implementation registry has implementations named `t_nohup` and `t_wait_for`, next to `t_execute_async` and `t_query_exec`.
- `t_nohup` takes `command` (string, required) and `timeout` (integer minutes, optional, default 10). It starts, with no delay, the shell command `timeout <timeout × 60> <command>` through the asynchronous start implementation, and returns exactly what that start returns (same JSON fields, inline output rule, FIFO stdin, completion event). E.g. `sleep 30` with the default becomes `timeout 600 sleep 30`; with `timeout` 3 it becomes `timeout 180 sleep 30`.
- The `timeout` value is truncated to an integer before converting to seconds.
- `t_wait_for` takes `howmuch` (number, required) and `unit` (string, optional, default `s`). Units: `s` = 1 s, `m` = 60 s, `h` = 3600 s, `d` = 86400 s. The wait is `howmuch × unit` seconds, capped at 3600 s per call.
  - It sleeps for the whole wait, then takes every pending completion event (non-blocking, until none remain) and returns the JSON object `{"waited_seconds": <wait, rounded to 3 decimals>, "completed": [...]}`.
  - Each `completed` entry has `tool_exec_id`, `returncode`, `duration_time`, `stdout_localfile`, `stderr_localfile`; when the execution is still known, also `command` (the full started command, including the `timeout` prefix), `stdout_last_lines` and `stderr_last_lines` (the same last-lines excerpt used by polling; empty string for empty output).
  - Completion events taken by `wait_for` are consumed: they are not reported again by a later `wait_for`. `nohup_query` still reports the execution's final state.
  - E.g. after `nohup` of `echo hi && sleep 1`, `wait_for(2, "s")` returns after ~2 s with `waited_seconds` 2 and one entry with `returncode` 0 and `stdout_last_lines` `hi`. With nothing pending, `wait_for(5, "m")` returns `{"waited_seconds": 300.0, "completed": []}`.
- Tool `nohup` (dispatched to `t_nohup`):
  - Parameters: `command` (string, "Shell command to run.", required) and `timeout` (integer, "Maximum minutes the command may run before being killed (default <N>).", optional).
  - Description: `Start a shell command asynchronously, like nohup(1). Returns tool_exec_id and local file paths for stdin (FIFO), stdout, and stderr. Write to stdin_localfile to send input to the running process. Execution is bounded: the command is killed after `timeout` minutes (default <N>). If the command finishes within 100 ms and both outputs are under 4096 bytes, stdout/stderr are inlined immediately.`
- Tool `nohup_query` (dispatched to `t_query_exec`):
  - Parameter: `tool_exec_id` (string, "The tool_exec_id returned by nohup.", required).
  - Description: `Poll a command started with nohup. When completed, includes returncode and inlines stdout/stderr if both are under 4096 bytes; otherwise reports file sizes. Polling the same still-running tool_exec_id twice in a row is denied: call wait_for(howmuch, unit) to let it finish instead.`
- Consecutive-poll denial: the process remembers the `tool_exec_id` of the last successful poll (one shared tracker, across sessions). A poll of that same id while its execution is still running is denied and returns, without polling, the JSON object `{"error": "denied: nohup_query was just called for this tool_exec_id and it is still running", "hint": "Use wait_for(howmuch, unit) to wait for it to finish instead of polling nohup_query again.", "tool_exec_id": "<id>"}`. A denied poll does not change the tracker, so further repeats stay denied.
  - The denial lifts when another id is polled (e.g. poll A, poll B, poll A → the third poll succeeds), when any new execution is started (the tracker is cleared), or when the execution has completed (a poll of a completed execution always returns its final state).
- Tool `wait_for` (dispatched to `t_wait_for`):
  - Parameters: `howmuch` (integer, "How long to wait, in the given unit.", required) and `unit` (string, enum `["d", "h", "m", "s"]`, "Time unit for howmuch: s=seconds, m=minutes, h=hours, d=days. Waits above 3600s are rejected; split long waits into several calls.", optional).
  - Description: `Wait howmuch * unit for background commands started with nohup to finish, instead of busy-polling with nohup_query. Reports every execution that completed while waiting: returncode, output file paths, and the last lines of stdout/stderr. Use nohup_query afterwards for executions still running.`
- `<N>` in the descriptions is the default bound the caller chose when enabling the tools (10 unless overridden). The definitions are listed in the order `nohup`, `nohup_query`, `wait_for`.
- Enabling the tools on an agent spec modifies the spec in place and returns it:
  - The tool definition list is the spec's existing `tool_specs`, else its `inferred_tool_schema`, else empty. The three definitions are appended, and the resulting list is stored under both `tool_specs` and `inferred_tool_schema`.
  - If the spec has a `tools` list (implementation names parallel to the definitions), `t_nohup`, `t_query_exec` and `t_wait_for` are appended to it, in that order, so it stays aligned with the definitions.
  - Otherwise `tool_dispatch` is created if missing, and gets `nohup` → `t_nohup`, `nohup_query` → `t_query_exec` and `wait_for` → `t_wait_for`, each with an empty parameter map.

## Edge cases
- Enabling is idempotent: if a tool named `nohup` is already among the definitions, the spec is left unchanged (no duplicate definitions, `tools` or dispatch entries).
- The default bound passed when enabling only changes the descriptions shown to the model; a call that omits `timeout` is still bounded at 10 minutes.
- When the bound expires, the command is terminated by `timeout`, so the completion event and poll report its exit status (124 for a normal timeout kill).
- `wait_for` validation happens before any sleep; each failure returns the JSON object `{"error": ...}` immediately:
  - unknown unit → `unknown unit '<unit>', expected one of d/h/m/s`;
  - a wait ≤ 0 (`howmuch` zero or negative) → `howmuch must be a positive number`;
  - a wait above 3600 s → `wait of <seconds>s exceeds the 3600s cap` (e.g. 3601 s, `2 h`, `1 d`). Exactly 3600 s (`1 h`) is accepted.
- Fractional `howmuch` values are accepted by the implementation (e.g. 0.01 s).
- The low-level `t_execute_async` / `t_query_exec` implementations remain available under their names and unbounded; the consecutive-poll denial belongs to `t_query_exec` itself, so it also applies to any tool dispatched to it, and starting an execution through `t_execute_async` also clears the tracker.
- An unknown `tool_exec_id` keeps its original `unknown tool_exec_id` error and is not subject to the denial.
