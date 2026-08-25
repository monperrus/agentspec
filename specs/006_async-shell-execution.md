# Asynchronous shell execution

The implementation registry offers two more built-in implementations that let a model run shell commands in the background: start a command (optionally after a delay of some minutes, so the model can schedule a check without busy-polling) and poll it later. When a command finishes almost at once with small output, the output comes back inline in the start result, which saves a polling round-trip. An agent spec refers to them in `tool_dispatch` by the implementation names `t_execute_async` and `t_query_exec`.

## Behaviour
- Start (`t_execute_async`, argument `command` (string, required) and `when` (integer minutes, optional, default 0)):
  - If `when` is non-zero, first blocks for `when` minutes; nothing is spawned and no execution id exists until the wait ends. With `when` 0 or absent it starts at once.
  - The capture directory is `~/.cache/async_agent_execs/<session_id>/`, where `<session_id>` is the id of the session whose tool call started the command. The session id is known to every tool invocation because it is recorded before each tool dispatch.
  - Runs the command string through the system shell in a new process group/session. Stdout and stderr go to the capture files `<capture dir>/<id>.stdout` and `<id>.stderr`. `<id>` is a new random 12-character lowercase hex execution id.
  - The process's stdin is a named FIFO created at `<capture dir>/<id>.stdin`. Bytes written to that path by anyone (the write-file tool, a shell redirect such as `echo yes > <path>`) are delivered to the running process's stdin.
  - The capture directory and its parents are created if missing. Capture files and FIFOs are never deleted.
  - Waits at most 100 ms for the command to exit, then returns a JSON object with `tool_exec_id`, `started_at`, `cwd`, `stdin_localfile`, `stdout_localfile` and `stderr_localfile` (absolute paths).
  - `started_at` is the local wall-clock time of the spawn, formatted `YYYY-MM-DDTHH:MM:SS` (ISO 8601, second precision, no fractional seconds and no timezone offset).
  - `cwd` is the absolute path of the agent process's current working directory at spawn time, which is the directory the command runs in.
  - If the command exited within that window, the object also has `completed: true`, `returncode` and `duration_time` (seconds since start, rounded to 3 decimals). It then also has `stdout` and/or `stderr`, each included only if that capture file is at most 4096 bytes.
  - Otherwise, the command keeps running in the background and the object has only the id, `started_at`, `cwd` and path fields above.
  - When such a background command later exits, a completion event is published to a process-wide completion queue that any caller can consume. The event holds `tool_exec_id`, `returncode`, the stdout and stderr capture paths, the working directory (`cwd`), and the duration in seconds (rounded to 3 decimals). A command that completed within the 100 ms window publishes no event, because its result was already returned inline.
- Poll (`t_query_exec`, argument `tool_exec_id`):
  - Returns a JSON object with `completed` (boolean), `returncode` (always present: `null` while the command is running, the integer exit status once completed), `started_at` (the same value the start result returned), `cwd` (the same working directory the start result returned), `duration_time` (seconds since start, rounded to 3 decimals), `stdin_localfile` (the FIFO path), `stdout_localfile_size` and `stderr_localfile_localsize` (current capture file sizes in bytes; the field names are exactly these).
  - When completed, it also has, under the same 4096-byte rule for each stream, `stdout` and/or `stderr`.
- Each implementation returns its text result together with metadata `{"result": <same text>}`.
- When a `read_file` tool call's `path`, after expanding a leading `~` to the home directory, is exactly the stdout or stderr capture file path of a command started in the current process, the displayed tool result is prefixed with a yellow line `  shell output from: <command>` (the label bold), followed by the usual dimmed result display. The result sent to the model and written to the session log is unchanged.
- There is no longer a standalone ready-made asynchronous coding agent; the grace-period coding agent is the ready-made agent that offers these implementations to the model.

## Edge cases
- When no session id is available (the start implementation is called outside a session's tool dispatch, or the session has no id), files go directly into `~/.cache/async_agent_execs/` with no subdirectory.
- The capture directory is chosen when the command is spawned, i.e. after any `when` delay.
- Output inlining is decided per stream: a small stdout is inlined even when stderr is too big, and vice versa. A stream whose capture file cannot be read is omitted.
- Polling an id that was not started in the current process returns `{"error": "unknown tool_exec_id: <id>"}`. Execution ids do not survive a process restart, even though their capture files stay on disk.
- A missing capture file is reported with size 0.
- A `read_file` path that is not a capture path of a known execution (another file, the stdin FIFO path, a relative spelling of the path, or an execution from an earlier process) gets no `shell output from:` line.
- A delayed start runs the command only after the full wait, and the call does not return during the wait. The 100 ms inline window and `duration_time` count from the actual spawn, not from the call. A delayed command still running 100 ms after spawn returns only its id and capture paths, and must be polled.
- The former implementation name `t_plan_delay` and tool `plan_shell_command` no longer exist; an agent spec that names `t_plan_delay` in `tool_dispatch` refers to an unknown implementation.
- Start and poll never wait for or kill a long-running command. It keeps running until it exits by itself.
- The agent keeps its own write end of the FIFO open while the process runs, so a writer closing the FIFO does not send end-of-file: a command that reads stdin until EOF (e.g. `cat`) keeps waiting for more input. The agent's write end is closed once the process exits.
- A write to the FIFO after the process has exited has no reader and fails or blocks, depending on how the writer opens it.
