# Asynchronous shell execution

The implementation registry offers three more built-in implementations that let a model run shell commands in the background: start a command, poll it later, and run a command after a planned number of minutes without busy-polling. When a command finishes almost at once with small output, the output comes back inline in the start result, which saves a polling round-trip. An agent spec refers to them in `tool_dispatch` by the implementation names `t_execute_async`, `t_query_exec` and `t_plan_delay`. A ready-made asynchronous coding agent offers them to the model together with file reading and writing.

## Behaviour
- Start (`t_execute_async`, argument `command`):
  - Runs the command string through the system shell in a new process group/session. Stdout and stderr go to the capture files `~/.cache/async_agent_execs/<id>.stdout` and `<id>.stderr`. `<id>` is a new random 12-character lowercase hex execution id.
  - The capture directory is created if it is missing. Capture files are never deleted.
  - Waits at most 100 ms for the command to exit, then returns a JSON object with `tool_exec_id`, `stdout_localfile` and `stderr_localfile` (absolute paths).
  - If the command exited within that window, the object also has `completed: true`, `returncode` and `duration_time` (seconds since start, rounded to 3 decimals). It then also has `stdout` and/or `stderr`, each included only if that capture file is at most 4096 bytes.
  - Otherwise, the command keeps running in the background and the object has only the three fields above.
- Poll (`t_query_exec`, argument `tool_exec_id`):
  - Returns a JSON object with `completed` (boolean), `duration_time` (seconds since start, rounded to 3 decimals), `stdout_localfile_size` and `stderr_localfile_localsize` (current capture file sizes in bytes; the field names are exactly these).
  - When completed, it also has `returncode` and, under the same 4096-byte rule for each stream, `stdout` and/or `stderr`.
- Planned command (`t_plan_delay`, arguments `command` (string) and `when` (integer), both required): blocks for `when` minutes, then starts `command` exactly as the start implementation does and returns the start result. The model can then re-plan from the outcome.
- Each implementation returns its text result together with metadata `{"result": <same text>}`.
- The asynchronous coding agent:
  - Is invoked as `<agent> MODEL [TASK...] [--endpoint URL] [--session SESSION_ID]`. MODEL is a model id or a `run://` URI. The endpoint defaults to the standard default endpoint. `--session` resumes that session.
  - With TASK words it runs one turn on the words joined by spaces, saves the message snapshot even if the turn fails, and exits. Without TASK it starts the REPL.
  - Uses an in-memory agent spec with `status: "default"` and structured tool calls, and offers five tools in this order: `execute_shell_command(command)`, `query_tool_exec(tool_exec_id)`, `plan_shell_command(command, when: integer)`, `read_file(path)`, `write_file(path, content)`. The last two map to the built-in read and write implementations. `plan_shell_command` is described to the model as: wait `when` minutes, then run `command` and return its result, to schedule a check on a long-running task without busy-polling.
  - Appends to the system prompt: "You are a coding agent. Start shell commands with execute_shell_command — they run in the background. Use query_tool_exec to poll status. Use plan_shell_command(when) to wait N minutes before re-checking long-running commands."
  - At startup prints (dimmed) `Model: <model>  |  tools: <comma-separated tool names>`, then `Session: <session id>`.

## Edge cases
- Output inlining is decided per stream: a small stdout is inlined even when stderr is too big, and vice versa. A stream whose capture file cannot be read is omitted.
- Polling an id that was not started in the current process returns `{"error": "unknown tool_exec_id: <id>"}`. Execution ids do not survive a process restart, even though their capture files stay on disk.
- A missing capture file is reported with size 0.
- The planned command runs only after the full wait. Its result follows the start rules, so a command that is still running after 100 ms returns only its id and capture paths, and must be polled.
- Start and poll never wait for or kill a long-running command. It keeps running until it exits by itself.
