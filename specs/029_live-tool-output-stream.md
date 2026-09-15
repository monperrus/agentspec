# Live tool output stream

The shell tool (`exec_shell`) and the search tool echo their subprocess output live, line by line, so a human watching the terminal sees progress. A library caller can choose where that echo goes. This lets a host that speaks a protocol on standard output (e.g. MCP, ACP, LSP) send the echo elsewhere, typically standard error, so it does not corrupt protocol frames.

## Behaviour
- Each line the subprocess writes to its stdout or stderr is written to the live output stream as soon as it is read, unchanged, and the stream is flushed after each line.
- By default the live output stream is the process's current standard output. The default is resolved at every write, not once at startup, so a caller that replaces the process-level standard output object later is still honoured.
- A library caller can set the live output stream to any writable text stream. From then on, all live echo from these tools goes only to that stream and nothing is echoed to standard output.
- A library caller can reset the stream to "none", which restores the default (the current standard output).
- A library caller can query the current live output stream. With no stream set, this returns the current standard output.
- The setting is process-wide and applies to all sessions and tool calls.
- Redirecting the echo does not change the tool result: the captured stdout, stderr, return code and metadata returned to the model are the same whatever the destination.

## Edge cases
- If writing to or flushing the live output stream fails for any reason (closed stream, broken pipe, a sink that raises), the failure is silently ignored. The tool still runs to completion and returns its full result. For example, `exec_shell` running `echo survives` with a failing sink returns output containing `survives` and return code 0.
