# Opt-in sandboxed tool execution (Bubblewrap)

A host program can supply a tool executor when it starts a task, a run or a single turn. When none is supplied, tool calls are dispatched locally in the controller process, as before. A provided Bubblewrap-based executor (Linux) confines the built-in file tools to a workspace directory and runs synchronous shell commands inside an unprivileged `bwrap` sandbox with no network, while the controller keeps the model credential and network access. Any tool the executor cannot isolate is rejected; it never falls back to local execution.

## Behaviour
- Executor contract: given the tool name, the model's arguments, the tool's dispatch entry (implementation name and parameter map) and a session context containing only the session id, the executor returns the result text sent to the model plus the data logged in the `tool_result` record. A caller can implement its own executor.
- A local executor reproducing the default in-controller dispatch is also available.
- An executor passed when starting a task or run applies to every tool call of the session; an executor passed for a single turn replaces the session's executor from that turn on.
- A sandbox policy has: `workspace` (a directory, resolved to an absolute real path), `network` (default `"none"`), `writable_paths` (workspace-relative, default `["."]`), `read_only_paths` (default empty), `timeout_seconds` (default 60) and `environment` (a string map, default absent).
- When the session's executor carries a policy, the `session_start` log record gains a `sandbox_policy` field holding these six fields, with `workspace` as a string; otherwise `sandbox_policy` is `null`.
- Creating the Bubblewrap executor requires a `bwrap` executable on `PATH` (the binary name is configurable); if it is missing, creation fails with an error stating that it refuses to fall back to local execution.
- Supported implementations are selected by the dispatch entry's implementation name: `t_read`, `t_write`, `t_update` (file tools) and `t_run` (synchronous shell). Arguments are renamed through the entry's parameter map before use.
- File tools run in the controller but only on paths inside the workspace:
  - `t_read` returns the file text; with `offset` and/or `limit` it returns only those lines (0-based offset, `limit` lines). An OS error yields `ERROR: <message>`.
  - `t_write` creates missing parent directories, writes `content` and returns `wrote <N> bytes` (N = length of the content).
  - `t_update` replaces every occurrence of `old` with `new` and returns `OK: replaced <N> occurrence(s) in <path>`; if `old` is absent it returns `ERROR: old text not found` and leaves the file unchanged.
- `t_run` runs `/bin/bash -c <command>` under `bwrap` with: a new session, network unshared, killed when the controller dies, fresh `/proc`, `/dev` and a tmpfs `/tmp`; `/usr`, `/bin`, `/lib`, `/lib64`, `/etc` bound read-only when they exist; the workspace mounted read-only at `/workspace`, then each writable path bound read-write at `/workspace/<path>` (`.` makes the whole workspace writable; other writable paths are created if missing); working directory `/workspace`.
- The sandboxed process inherits no environment from the controller; it receives exactly the policy's `environment`, or only `PATH=/usr/bin:/bin` when none is set.
- The shell result is compact JSON (no spaces) `{"stdout":…,"stderr":…,"returncode":…}`; the logged data contains these fields plus `result`.
- On timeout the command is killed and the result is `{"stdout":…,"stderr":…,"error":"command timed out after <timeout_seconds> s"}` with whatever output was captured.

## Edge cases
- A `network` other than `"none"`, a non-positive `timeout_seconds` or a `workspace` that is not an existing directory is rejected when the policy is created.
- Paths must be non-empty strings; absolute paths are rejected ("absolute paths are not allowed in a sandbox"); a path that resolves outside the workspace, including through `..` or a symlink, is rejected ("path escapes the sandbox workspace"). This also applies to `writable_paths`.
- A non-string `command` is rejected.
- Any other implementation (custom tools, asynchronous shell tools) makes the call fail with an error naming the tool and stating that it is not supported and that custom and async tools must provide a sandbox adapter.
- `read_only_paths` is recorded in the policy but grants no extra mounts.
- Optional shell rewriting through `rtk` does not apply to commands run by the Bubblewrap executor.
