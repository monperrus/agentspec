# Lifecycle hooks

The agent runs user-configured hooks at fixed points in the session lifecycle. It implements the core that the Claude Code and Codex CLI hook conventions share, so a `hooks.json` written for either tool works unchanged: the same config shape, matcher semantics, JSON input on stdin, and exit-code plus JSON-on-stdout output contract. A hook is either an external command or an in-process function registered by a programmatic caller. Both kinds go through one interpretation of the output contract, so a function and a script with the same logic produce identical decisions. Hooks are the control plane: they may block or rewrite. The event subscription stream stays the observation plane.

## Behaviour
### Configuration
- Config shape: `{"hooks": {"<Event>": [{"matcher": "<m>", "hooks": [<handler>, …]}, …]}}`. An inline source may also be the bare event map (`{"<Event>": [...]}`) without the outer `hooks` key.
- A handler is `{"type": "command", "command": "<cmd>", …}`. When `type` is missing and `command` is present, the type is `command`. Optional fields:
  - `args`: a list. When present, the command is run in exec form: `command` is resolved on `PATH` and spawned directly with `args`, without a shell. Otherwise `command` is run through the shell.
  - `commandWindows`: replaces `command` on Windows.
  - `timeout`: seconds, a non-negative number.
  - `async`: boolean, default false.
  - `statusMessage`: a string.
  - `additionalContextLimit`: a non-negative integer.
  - A field with an invalid value is ignored.
- Handler types `mcp_tool`, `prompt` and `agent` are parsed but skipped with a warning. Any other non-`command` type, or a missing or empty `command`, is skipped with a warning.
- Sources merge additively, in this order:
  1. the programmatic `hooks` option of a session or one-shot task (a file path, a directory path, an inline config, or a list of any of these), which the CLI fills from `--hooks PATH` (repeatable);
  2. the agent spec's `behaviour.hooks` (same accepted forms);
  3. the project layer, inside the agent's per-project config directory at the root of the enclosing git work tree (or the current directory when outside a git repository): first its `hooks.json` file, then its `hooks/` directory;
  4. the user layer, inside the agent's per-user config directory in the home directory: first its `hooks.json` file, then its `hooks/` directory.
- A path source that is a directory is scanned as a hooks directory (see below). Any other path is parsed as a `hooks.json`-shaped file.
- A leading `~` in a path is expanded. A missing file or directory is silently skipped. An unreadable file, invalid JSON, a non-object config, an unknown event, or a malformed matcher group or handler prints a warning `⚠ hooks: <message>` to stderr at startup and skips only the bad part. It is never fatal.
- After all sources are merged, duplicates are removed. Two command hooks are one hook when they have the same event, the same matcher, and commands that resolve to the same file (after `~` expansion and resolution to an absolute, symlink-free path). The first occurrence, from the earlier layer, is kept. A command containing any of the characters `` ;|&<>$`*?(){}[] `` or a newline is a shell one-liner and is never deduplicated. In-process function hooks are never deduplicated.
- A master switch turns off every hook, whatever its source: the top-level spec key `hooks_enabled` (default `true`), a programmatic override (which wins over the spec), and the CLI flag `--no-hooks`.
- The `hooks` option and the `hooks_enabled` override are accepted in every launch mode: a one-shot task, a programmatic session, and both interactive REPL variants (the plain one and the one with type-ahead input). The CLI forwards `--hooks` and `--no-hooks` to whichever mode it starts.
- A programmatic caller can register a hook on a live session, giving an event, an optional matcher, an optional timeout, an async flag, and either a function or a command (with optional exec-form args). It can also merge a further config source into a live session, and gets back the added entries and the warnings. Registering an unknown event, or giving neither a function nor a command, is an error.
- When a session is restored in memory, it keeps its hooks. A `hooks` option given at restore time is merged in, and a `hooks_enabled` override replaces the stored value (default enabled).

### Hooks directory
- In a hooks directory, the presence of an executable file is its registration. No JSON is needed.
- Accepted layout:
  - `<dir>/<script>`: the event is inferred from the file name, and the matcher is empty.
  - `<dir>/<event>/<script>`: the directory names the event, and the matcher is empty.
  - `<dir>/<event>/<matcher>/<script>`: the second directory's name is the matcher, taken verbatim.
- An event is inferred from a name as follows. The name is normalized: lowercased, with every character other than a letter or digit removed. For a file, the name is taken without its last extension. An exact match against a normalized known event name (including the events that never fire) wins. Otherwise, the longest known event name that the normalized name ends with is used. So `pre-compact.sh` → `PreCompact`, `notify-stop.py` → `Stop`, and `notify-subagent-stop.py` → `SubagentStop`.
- A discovered script runs in exec form with no arguments: its path is spawned directly, never through a shell. It follows the usual input and output contract.
- These entries are ignored silently, at any level: names starting with `.`, names ending in `.disabled`, and the names `__pycache__`, `readme`, `readme.md` and `notes` (case-insensitive).
- These entries produce a startup warning and are not registered:
  - a file at the top level whose event cannot be inferred;
  - a top-level directory that is not an event name;
  - a directory below `<event>/<matcher>/` (nested too deep);
  - a file that is not executable by the current user.
- Entries are discovered in sorted name order, so the result is deterministic.
- A programmatic caller can scan a directory on its own and get back the discovered entries and the warnings.

### Events
- Events that fire: `SessionStart`, `SessionEnd`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `PreCompact`, `PostCompact`, `Interrupt`.
- `PermissionRequest`, `SubagentStart`, `SubagentStop` and `Notification` load without warnings but never fire.

### Matchers
- `""`, `"*"` or an absent matcher matches everything.
- A matcher made only of letters, digits, `_`, `-`, spaces, `,` and `|` is a list of exact alternatives, split on `|` or `,` and trimmed.
- Any other matcher is an unanchored regular expression. An invalid regex matches nothing.
- The matched values per event:
  - tool events: the canonical tool name and the native tool name (a hit on either one counts);
  - `SessionStart`: `startup` or `resume`;
  - `PreCompact`/`PostCompact`: `manual` or `auto`;
  - `SessionEnd`: its reason.
  - `UserPromptSubmit`, `Stop` and `Interrupt` ignore the matcher.
- Canonical tool names: `exec_shell`, `nohup`, `nohup_wait`, `nohup_query` → `Bash`; `read_file` → `Read`; `write_file` → `Write`; `str_replace`, `update`, `update_file` → `Edit`; `search`, `search_files` → `Grep`; `glob`, `find_files` → `Glob`; `list_dir`, `list_directory` → `LS`. Any other tool keeps its own name.

### Input
- The hook receives one JSON object: on stdin for a command, as the argument for a function. The object always has these fields:
  - `session_id`
  - `transcript_path` (the session log path, or `null`)
  - `cwd`
  - `hook_event_name`
  - `model`
  - `permission_mode` (`dontAsk` when non-interactive, else `default`)
  - `scratchpad_dir`
  - `prompt_id` (a fresh id per turn, `null` before the first turn)
- Per-event fields:
  - tool events: `tool_name` (canonical), `tool_input` (the arguments), `tool_use_id`, and a field that carries the native tool name. `PostToolUse` adds `tool_response`, the result text.
  - `SessionStart`: `source`.
  - `UserPromptSubmit`: `prompt`.
  - `Stop`: `stop_hook_active` and `last_assistant_message`.
  - `PreCompact`/`PostCompact`: `trigger`.
  - `SessionEnd`: `reason`.
- A command runs with the session's working directory as its cwd.

### Output contract
The same rules apply to every hook:
- Exit 0 with empty stdout: no decision.
- Exit 0 with a JSON object on stdout: structured control.
  - Common fields:
    - `continue: false`, with an optional `stopReason`: stop.
    - `systemMessage`: shown to the user.
    - `decision: "block"` with an optional `reason`: block. `"approve"` is accepted, and means allow only on `PreToolUse`.
    - `suppressOutput` is accepted and ignored.
  - `hookSpecificOutput` must carry `hookEventName` equal to the event. It may contain:
    - `permissionDecision` (`allow`, `deny` or `ask`) and `permissionDecisionReason`: `PreToolUse` only.
    - `updatedInput` (an object): `PreToolUse` only, and only together with `allow` or `ask`.
    - `updatedToolOutput` (a string): `PostToolUse` only.
    - `additionalContext` (a string): every firing event except `SessionEnd` and `Interrupt`.
  - A field used on the wrong event, a wrong type, a mismatched `hookEventName`, `continue` on `PreToolUse`, or an unknown `decision` value makes the whole output invalid. That is a non-blocking error.
- Exit 0 with plain-text stdout: for `SessionStart` and `UserPromptSubmit`, the text becomes additional context. For `Stop` it is an error ("expects JSON on stdout"). For all other events it is ignored.
- Exit 2 means blocking, on the events that can block: `PreToolUse`, `UserPromptSubmit`, `PostToolUse`, `Stop`, `PreCompact`. The reason is taken from the JSON `reason`, else `permissionDecisionReason`, else the trimmed stderr, else `blocked by hook`. On `PreToolUse`, exit 2 also means deny. On the other events, exit 2 only shows stderr to the user.
- Any other exit code, a timeout, a failure to start, or stdout that is JSON but not an object is a non-blocking error (`hook exited with code <n>: <stderr>`, `hook timed out after <t>s`, …). The operation goes ahead (fail open).
- An in-process function follows the same contract. Returning nothing or an empty mapping is exit 0 silent. Returning a mapping is that JSON on stdout. Returning a string is plain stdout. Signalling a block with a message is exit 2 with that message as stderr. Any other failure, or a return value of another type, is exit 1.
- Timeouts: 60 s by default. `SessionEnd` and `Interrupt` default to 1 s and are capped at 3 s.

### Dispatch and combination
- All matching non-async handlers run concurrently. Before they run, one `hook_status` event is emitted per handler, carrying `text` (its `statusMessage`, else a label with the event, the matcher and the handler), `source` and `fmt`.
- Async handlers run in the background and their decisions are discarded. Their additional context is delivered at the next safe point: the next tool-result suffix, or the system prompt at startup.
- Combination rules:
  - Errors are joined with `; `. System messages and additional contexts are joined with newlines, and all of them are delivered.
  - Any `continue: false` wins over every other decision.
  - Permission decisions follow the precedence `deny` > `ask` > `allow`. The first reason and the first `updatedInput` among the winners are used.
  - Any block wins over no block, with the first non-empty reason.
  - The first `updatedToolOutput` is used.
- Model-visible additional context from one handler is capped at 10,000 characters, or its `additionalContextLimit` (0 means no cap). When the text is longer, the full text is saved as `hook_outputs/<id>.txt` in the session directory (or the log's directory). The model sees the first 4000 characters, then `\n\n…[<N> chars omitted; full hook output saved to <path>]\n\n`, then the last 2000 characters. If the file cannot be saved, the marker reads `…[<N> chars omitted from hook output]`.
- An error is reported as a `hook_warning` event (`text`, formatted `⚠ hook error: <error>` in yellow) and as a `hook_error` log record (`event`, `error`, `ts`). A system message is reported as a `hook_warning` event formatted `[hook] <message>`.
- Additional context from events in the middle of a turn is queued. It is appended, after a blank line, to the next tool result in the same way as the token-awareness countdown and after it.

### Lifecycle points
- `SessionStart` fires once at session creation, after the startup log record, with source `resume` for a resumed session and `startup` otherwise. Its additional context is appended to the system prompt after a blank line. It cannot block.
- `UserPromptSubmit` fires at the start of a turn that has a task, before anything is journaled or sent. A block or stop rejects the prompt. Nothing goes to the model, a `user_prompt_blocked` log record is written, a `hook_warning` event is emitted, and the turn's final reply is `Prompt rejected by hook: <reason>` (the reason is the stop reason, else the block reason, else `prompt blocked by UserPromptSubmit hook`).
- `PreToolUse` fires before the tool call's write-ahead journal record.
  - Deny or block: the tool does not run. The result is `ERROR: tool call denied by hook: <reason>`, where the reason is the block reason, else `permissionDecisionReason`, else `tool call blocked by PreToolUse hook`. The journal gets `tool_start` and then `tool_end` with `outcome: "hook_deny"`. `tool_call` and `tool_result` events are emitted, and the log record carries `hook: "PreToolUse:deny"`.
  - `updatedInput` replaces the whole argument object before the call is journaled and dispatched. `file_path` → `path`, `old_string` → `old_str` and `new_string` → `new_str` are renamed; `description`, `timeout` and `run_in_background` are dropped; other keys pass through unchanged. A `tool_call_rewritten` log record is written. The `tool_call` event shows the rewritten arguments.
  - `ask` in an interactive session: before dispatch, the user is prompted `? <permissionDecisionReason or "hook asks for confirmation before <tool>"> [y/N] `, with type-ahead input paused. Any answer except `y`/`yes` (including EOF or Ctrl-C) refuses. The result is then `ERROR: tool call denied by user (PreToolUse hook asked)`, the journal gets `tool_end` with `outcome: "hook_ask_deny"`, and the log record carries `hook: "PreToolUse:ask-deny"`. In a non-interactive session, `ask` does not prompt and the call runs.
- `PostToolUse` fires after the tool has run, before its `tool_end` journal record. `updatedToolOutput` replaces the result. A block replaces the result with `ERROR: PostToolUse hook rejected this tool result: <reason>` (default reason `tool output rejected by PostToolUse hook`). The replaced text is what the model sees and what gets journaled.
- `Stop` fires when the model gives a final reply, with `stop_hook_active: false`. A block refuses the stop: the reason is appended as a new user message, the guard flag is set, and the turn loop continues. While the flag is set, `Stop` still fires with `stop_hook_active: true`, but a block is ignored. The flag is reset at the start of every turn.
- `PreCompact` fires before a compaction attempt, with trigger `manual` when no prompt size was given and no watermark is recorded, and `auto` otherwise. A block or stop skips the compaction. It emits `compaction_skipped` (reason `PreCompact hook`) and leaves the watermark unchanged. `PostCompact` fires after a successful compaction with the same trigger.
- `SessionEnd` fires after the final snapshot and session-end log record. Its reason is `prompt_input_exit` when the REPL exits, `clear` on `/clear`, and `other` after a one-shot task (programmatic, CLI or stdin task).
- `Interrupt` fires when Ctrl-C interrupts a running turn, before the running subprocess is killed. It is advisory and cannot prevent the interrupt.

## Edge cases
- Starting the interactive REPL, with or without `--hooks`/`--no-hooks`, never fails because of the hook options. The given hooks apply to the REPL session, and `--no-hooks` disables them there.
- Hooks disabled, or no hooks configured: no hook runs, no hook event is emitted, and the behaviour is exactly as without this feature.
- A matcher `Bash` matches a call to `exec_shell`. A matcher `exec_shell` matches it too.
- A matcher `mcp__.*` is a regex. `Edit|Write` matches either name exactly, and `Edi` matches neither.
- A hook exits 2 on `PostCompact` with stderr `x`: nothing is blocked, and `x` is shown to the user as a system message.
- A hook exits 1 on `PreToolUse`: the tool runs, and a `hook_warning` reports `hook exited with code 1: <stderr>`.
- Two `PreToolUse` hooks return `allow` and `deny`: the call is denied.
- `updatedInput` sent with `permissionDecision: "deny"` or with no permission decision: the output is invalid, and a non-blocking error is reported.
- A `Stop` hook that blocks every time makes the turn continue exactly once. The second final reply ends the turn.
- A settings file configuring only never-firing events loads without warnings, and its hooks never run.
- `hooks/helpers.sh` (no event in the name) or a non-executable `hooks/Stop/notify.py`: a startup warning, and nothing is registered. Such a file never fails at dispatch time with exit 127.
- `hooks/Stop/my hook.sh`: the path with a space runs correctly, since no shell is involved.
- A `hooks.json` entry whose command is `<layer>/hooks/./notify-stop.py`, plus the discovered `<layer>/hooks/notify-stop.py`: the script runs once per `Stop`.
- The same script registered on `PreToolUse`/`Bash`, `PreToolUse`/`Edit` and `Stop`: three distinct hooks. The same shell one-liner listed twice: it runs twice.
