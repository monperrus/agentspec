# Grace-period coding agent

A ready-made coding agent that behaves like the asynchronous coding agent, but pauses for a 2-second grace period before starting each shell command. During that pause the operator can abort the command, either with Ctrl-C or through a cancellation request from another part of the program, such as a supervising model. It also offers a substring-replacement edit tool.

## Behaviour
- Invoked as `<agent> MODEL [TASK...] [--endpoint URL] [--session SESSION_ID]`, with the same one-shot and REPL modes, REPL behaviour and automatic completion turns as the asynchronous coding agent. The REPL banner is `secondguess-agent <MODEL>  (type 'exit' to quit)`.
- Offers five tools in this order: `exec_shell(command, when?: integer)`, `query_tool_exec(tool_exec_id)`, `read_file(path)`, `write_file(path, content)`, `str_replace_edit(path, old_str, new_str)`. All `str_replace_edit` parameters are required. It maps to the built-in substring-update implementation. `query_tool_exec`, `read_file` and `write_file` map to the built-in poll, read and write implementations.
- `exec_shell`:
  - If `when` is non-zero, first waits `when` minutes.
  - Then runs a 2-second grace period and checks for cancellation every 100 ms.
  - If no cancellation happened, it starts the command exactly like the asynchronous start implementation with no delay and returns that result.
  - If it was cancelled, the command is never spawned and the tool returns `{"cancelled": true, "command": "<command>", "message": "Command was cancelled during the 2.0s grace period."}`.
- Cancellation sources: Ctrl-C during the grace period, or a cancellation request from another thread. A request made before the grace period starts (for example, while the model is still generating the tool call) cancels the next command at once.
- The cancellation request is cleared at the end of every grace period, whether or not it caused a cancellation. A request therefore affects at most one command.
- The system prompt supplement is the asynchronous coding agent's supplement followed by a blank line and: "**Grace period**: every exec_shell call pauses 2.0s before actually starting the command. During this window the operator can Ctrl-C to abort. The command only begins after the grace period expires."
- The model sees the same `exec_shell` description as in the asynchronous agent, with two additions: it starts with a 2.0s grace period, and an **IMPORTANT** note says the command does not start immediately and can be cancelled during the grace period with Ctrl-C or by a supervisor.
- `query_tool_exec` is described as polling a command started with `exec_shell` (its `tool_exec_id` is described as "The ID returned by exec_shell."). When the command has completed, the result includes the return code and inlines stdout/stderr if both are under 4096 bytes; otherwise it reports file sizes.

## Edge cases
- The `when` delay cannot be cancelled. Only the grace period that follows it can.
- A cancelled command has no execution id, no capture files and no completion event.
