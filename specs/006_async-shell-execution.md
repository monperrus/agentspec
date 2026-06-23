# Asynchronous shell execution

The implementation registry offers three more built-in implementations that let a model run shell commands in the background: start a command, poll it later, and run a command after a planned number of minutes without busy-polling. When a command finishes almost at once with small output, the output comes back inline in the start result, which saves a polling round-trip. An agent spec refers to them in `tool_dispatch` by the implementation names `t_execute_async`, `t_query_exec` and `t_plan_delay`. A ready-made asynchronous coding agent offers them to the model together with file reading and writing.

## Behaviour
- Start (`t_execute_async`, argument `command`):
  - Runs the command string through the system shell in a new process group/session. Stdout and stderr go to the capture files `~/.cache/async_agent_execs/<id>.stdout` and `<id>.stderr`. `<id>` is a new random 12-character lowercase hex execution id.
  - The process's stdin is a named FIFO created at `~/.cache/async_agent_execs/<id>.stdin`. Bytes written to that path by anyone (the write-file tool, a shell redirect such as `echo yes > <path>`) are delivered to the running process's stdin.
  - The capture directory is created if it is missing. Capture files and FIFOs are never deleted.
  - Waits at most 100 ms for the command to exit, then returns a JSON object with `tool_exec_id`, `stdin_localfile`, `stdout_localfile` and `stderr_localfile` (absolute paths).
  - If the command exited within that window, the object also has `completed: true`, `returncode` and `duration_time` (seconds since start, rounded to 3 decimals). It then also has `stdout` and/or `stderr`, each included only if that capture file is at most 4096 bytes.
  - Otherwise, the command keeps running in the background and the object has only the four path/id fields above.
- Poll (`t_query_exec`, argument `tool_exec_id`):
  - Returns a JSON object with `completed` (boolean), `duration_time` (seconds since start, rounded to 3 decimals), `stdin_localfile` (the FIFO path), `stdout_localfile_size` and `stderr_localfile_localsize` (current capture file sizes in bytes; the field names are exactly these).
  - When completed, it also has `returncode` and, under the same 4096-byte rule for each stream, `stdout` and/or `stderr`.
- Planned command (`t_plan_delay`, arguments `command` (string) and `when` (integer), both required): blocks for `when` minutes, then starts `command` exactly as the start implementation does and returns the start result. The model can then re-plan from the outcome.
- Each implementation returns its text result together with metadata `{"result": <same text>}`.
- The asynchronous coding agent:
  - Is invoked as `<agent> MODEL [TASK...] [--endpoint URL] [--session SESSION_ID]`. MODEL is a model id or a `run://` URI. The endpoint defaults to the standard default endpoint. `--session` resumes that session.
  - With TASK words it runs the standard one-shot task mode on the words joined by spaces and exits. Without TASK it starts the standard REPL. Both modes get the `--session` value and the system prompt supplement below, and both print the standard startup lines of those modes; the agent prints no startup banner of its own.
  - Uses an in-memory agent spec with `status: "default"` and structured tool calls, and offers five tools in this order: `execute_shell_command(command)`, `query_tool_exec(tool_exec_id)`, `plan_shell_command(command, when: integer)`, `read_file(path)`, `write_file(path, content)`. The last two map to the built-in read and write implementations. `plan_shell_command` is described to the model as: wait `when` minutes, then run `command` and return its result, to schedule a check on a long-running task without busy-polling.
  - Appends to the system prompt: "You are a coding agent. Start shell commands with execute_shell_command — they run in the background. Use query_tool_exec to poll status. Use plan_shell_command(when) to wait N minutes before re-checking long-running commands."
  - `execute_shell_command` is described to the model as: start a shell command asynchronously; it returns `tool_exec_id` and local file paths for stdin (FIFO), stdout and stderr; write to `stdin_localfile` to send input to the running process; if the command finishes within 100 ms and both outputs are under 4096 bytes, stdout/stderr are inlined immediately.

## Edge cases
- Output inlining is decided per stream: a small stdout is inlined even when stderr is too big, and vice versa. A stream whose capture file cannot be read is omitted.
- Polling an id that was not started in the current process returns `{"error": "unknown tool_exec_id: <id>"}`. Execution ids do not survive a process restart, even though their capture files stay on disk.
- A missing capture file is reported with size 0.
- The planned command runs only after the full wait. Its result follows the start rules, so a command that is still running after 100 ms returns only its id and capture paths, and must be polled.
- Start and poll never wait for or kill a long-running command. It keeps running until it exits by itself.
- The agent keeps its own write end of the FIFO open while the process runs, so a writer closing the FIFO does not send end-of-file: a command that reads stdin until EOF (e.g. `cat`) keeps waiting for more input. The agent's write end is closed once the process exits.
- A write to the FIFO after the process has exited has no reader and fails or blocks, depending on how the writer opens it.
