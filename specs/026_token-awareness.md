# Model-facing token awareness

The agent puts the true token count into the model's own context, so the model can pace itself and write down its progress before automatic compaction. The system prompt declares a token budget. After model calls, a countdown is appended to the next tool result. When the remaining headroom crosses below a threshold, a one-time checkpoint reminder is appended too. Every number comes from the prompt-token counts that the server reports; nothing is estimated, padded or made up. The feature is on by default.

## Behaviour
- Configuration comes from four agent spec fields. A programmatic caller can override each one per session and for single tasks. An explicit override wins over the spec field, and the spec field wins over the default:
  - `token_awareness_enabled` (default `true`): master switch. When false, none of the injections below happens and no `token_budget` event is emitted.
  - `token_awareness_budget_tokens`: the countdown denominator B. Resolution order: explicit override → spec `token_awareness_budget_tokens` → spec `context_window` → the session's effective `compaction_trigger_tokens` (override → spec → `100000`). A missing or zero value at one step falls through to the next.
  - `token_awareness_reminder_tokens` (default `6144`): the reminder threshold T.
  - `token_awareness_update_every` (default `1`): the countdown is injected only after every N-th counted model call. Values below 1 are treated as 1.
- The spec field `context_window` (integer, optional) declares the model's context window in tokens. It currently only serves as the budget fallback. A value above the compaction trigger does not delay compaction.
- CLI flag `--context-window N` (integer, default unset) sets or overrides `context_window`. It is applied after the spec has been loaded and after a resumed session has been bound to its endpoint, and before the spec is validated. So it works on top of a spec file and on top of the default in-memory spec for `run://` models, and no spec file is needed. When the flag is absent, the loaded spec's value (or its absence) is kept.
- System prompt: when enabled, a newly created session appends this block to the system prompt after the environment block, separated from it by a blank line (`<B>` is the budget in decimal digits; `<agent>` is the agent's own fixed product name):
  ```
  <budget:token_budget><B></budget:token_budget>
  Your context window holds <B> tokens; the usage counter after each tool call shows how full it currently is. When it fills, <agent> automatically compacts older context into a summary and you continue in fresh space — this is normal operation, not a deadline. Do not stop tasks early or take shortcuts due to context capacity; there is always enough room to finish properly.
  ```
- Countdown: for each model response that carries a usage block (which increments the session's counted-call number C), with P = `usage.prompt_tokens`:
  - Nothing is produced when the feature is off, P ≤ 0 (missing counts as 0), B ≤ 0, or C is not a multiple of N.
  - Otherwise R = max(0, B − P) and the countdown text is `<system_warning>Token usage: <P>/<B>; <R> remaining</system_warning>`, with plain decimal numbers.
  - Each response's result replaces any countdown still pending from an earlier response in the same turn. A skipped response clears it.
- Placement: the pending text is appended, after a blank line (`\n\n`), to the content of the next tool result produced in the same turn, and is then consumed. Only that one result gets it; later results from the same response are unchanged. With native tool calls, this is the first `tool` message. With inline tool calls, it goes inside the first `[<name>] <result>` block of the `Tool results:` user message. No extra message is ever created. A response that ends the turn with a plain-text reply has no following tool result, so its countdown is dropped.
- Checkpoint reminder (edge-triggered): the session remembers the last R it computed (initially none). When R < T and the previous R was none or ≥ T, the countdown text is followed by a newline and then:
  ```
  <context_window_reminder>
  Your context window is nearly full; <R> tokens remain before compaction. Write concise progress notes in your next reply — goal, decisions, progress, learnings, next steps — then keep working. <Agent> will compact earlier context into a summary and you continue in fresh space. Do not stop the task; compaction is normal operation, not a deadline.
  </context_window_reminder>
  ```
  `<Agent>` is the product name with its first letter capitalised. After R rises back to ≥ T (for example after compaction shrinks the prompt), the next crossing fires the reminder again.
- `token_budget` event: emitted whenever a countdown is produced, with `used` (P), `budget` (B), `remaining` (R), `below_reminder_threshold` (R < T) and `fmt`, a dim magenta line `[budget] <R>/<B> tokens remaining`, with thousands separators and ` (below reminder threshold)` appended when below. It is emitted before the response's `usage` event.
- Usage log: when the feature is enabled, each per-response `usage` log record gains `token_budget_remaining` = max(0, B − P). This happens on every counted call, whatever N is.
- Snapshot: the message snapshot `metadata` gains `token_awareness`, an object with `enabled` (boolean), `budget_tokens`, `reminder_tokens` and `update_every`.
- In-memory restore: a restored session that predates this feature gets the defaults: enabled, B = its `compaction_trigger_tokens` (or `100000`), T = `6144`, N = `1`, no previous R. Explicit overrides given at restore time replace the stored values. The stored system prompt is not rewritten.

## Edge cases
- Default configuration: the system prompt declares `<budget:token_budget>100000</budget:token_budget>`. A call reporting 35000 prompt tokens followed by a tool call gives that tool result `<system_warning>Token usage: 35000/100000; 65000 remaining</system_warning>`.
- Default configuration, calls reporting 95000 then 96000, each followed by a tool call: the first tool result gets the countdown plus the reminder (`5000 tokens remain before compaction`). The second gets only the countdown.
- B = 10000, T = 2000, calls reporting 9000, 3000, 8500: the reminder fires on the first and third. The second shows `Token usage: 3000/10000; 7000 remaining` and no reminder.
- N = 3: only the tool result after the 3rd counted call (then the 6th, …) gets the countdown. Skipped calls neither update the remembered R nor emit an event.
- R = T exactly is not below the threshold.
- P > B: R is 0 and it counts as below the threshold.
- A response without a usage block produces nothing and does not advance C.
- `--context-window 922000` with a spec declaring `context_window` 128000 and no explicit budget: B is 922000.
- `--context-window` with a value ≤ 0 (for example `0`) makes the process exit with a non-zero status and an error message saying the value must be a positive integer. This happens before any session starts. A value that is not an integer is rejected as a usage error by the argument parser.
- Disabled: no budget block in the system prompt, no `<system_warning>`, no reminder and no `token_budget_remaining` field.
