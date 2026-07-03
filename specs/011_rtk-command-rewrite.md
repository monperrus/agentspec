# Optional shell command rewriting through rtk

A host program can opt in, once and before running any task, to route every shell command through the external `rtk` CLI proxy (https://github.com/rtk-ai/rtk), which rewrites commands so their output uses far fewer tokens. The feature is off by default and does nothing when `rtk` is not installed.

## Behaviour
- Opting in is a single library call with no arguments and no return value. It must be made before the agent starts dispatching tool calls to take effect.
- When opted in and an executable named `rtk` is found on `PATH`, the synchronous shell implementation (`t_run`) and the asynchronous start implementation (`t_execute_async`) rewrite their `command` argument before running it. All other arguments pass through unchanged.
- Rewriting runs `rtk rewrite <command>` (the command string as a single argument) with a 2-second timeout and captures its stdout.
- If `rtk` exits with status 0 or 3 and its stdout is non-blank, the whitespace-stripped stdout replaces the command. Otherwise the original command runs unchanged.
- The tool results, schemas and names seen by the model are unchanged; only the command actually executed differs.
- Without opting in, commands are never rewritten.

## Edge cases
- If `rtk` is not on `PATH` at opt-in time, the call is a no-op, and installing `rtk` later has no effect until opting in again.
- A timeout, a failure to start `rtk`, any other exit status, or blank output all fall back to the original command silently; no error reaches the model.
- Other implementations (e.g. polling a background command, file reading and writing) are not affected.
- Opting in more than once wraps the implementations again, so each command is passed through `rtk rewrite` once per opt-in call.
