# Context compaction

Long sessions compact automatically. When a model call reports that its prompt reached a configurable token threshold, the agent first asks the model to write down the state it needs to keep working, then asks it to summarize the older part of the conversation into a summary aimed at continuing the task. That summary replaces the older messages. The most recent messages stay as they were. Compaction is on by default. A policy decides when it runs, and it can also be triggered on demand.

## Behaviour
- Configuration comes from these agent spec fields. A programmatic caller can override each one per session, for single tasks, the sync REPL and the async REPL. An explicit override wins over the spec field, and the spec field wins over the default:
  - `compaction_enabled` (default `true`).
  - `compaction_trigger_tokens` (default `100000`): the prompt-token threshold.
  - `compaction_target_tokens` (default `20000`): the output-token limit (`max_tokens`) for the summary call.
  - `compaction_keep_last_turns` (default `2`): how many of the most recent non-system messages stay unchanged (subject to boundary snapping, below).
  - `compaction_policy` (default `"threshold"`): when automatic compaction runs. One of `"threshold"`, `"every_turn"` or `"never"`. A programmatic caller may instead supply a custom decision function.
  - `compaction_min_chars` (default `0`, meaning no minimum): compaction is skipped while the compactable text is shorter than this many characters.
- Automatic compaction is considered at two phases of a turn: `mid_turn` (after each main model call's usage has been recorded, inside the tool loop) and `turn_end` (after a successful final answer, once the `session_usage` event has been emitted).
- Policies:
  - `threshold`: at `mid_turn`, the threshold check below. Nothing at `turn_end`.
  - `every_turn`: at `mid_turn`, the same threshold check, as an overflow backstop. At `turn_end`, compaction is attempted unconditionally, whatever the token count and whatever `compaction_enabled` says. This attempt does not change the watermark W.
  - `never`: no automatic compaction in either phase, whatever `compaction_enabled` says.
  - Custom decision function: called at each phase with the session, the latest usage and the phase name (`mid_turn` or `turn_end`). Compaction is attempted whenever it returns true. It ignores `compaction_enabled`, the threshold and the watermark.
- The turn's result (including its final reply) is captured before `turn_end` compaction, so it stays intact even if compaction summarizes the whole history.
- Threshold check: triggering relies only on usage metadata from the server. The agent never counts tokens locally. At `mid_turn`, compaction runs when it is enabled and the response's `usage.prompt_tokens` is ≥ `compaction_trigger_tokens` and also > W + H (see hysteresis below). A missing value counts as 0. This check comes before the session token-budget check.
- Hysteresis: the session keeps a watermark W, which starts at 0. Each time compaction is attempted, W becomes the prompt-token count that triggered the attempt, which is the count before compaction. H = `compaction_trigger_tokens` ÷ 4, rounded down. Compaction can therefore fire again only once the prompt count has grown by more than a quarter of the threshold past the last triggering count. This stops a "compaction storm": the remaining context (system, summary and kept messages) can still sit at or above the threshold, and without the watermark every following turn would compact again.
- `/clear` resets W to 0.
- Split: the kept suffix begins at the K-th most recent non-system message, where K = `compaction_keep_last_turns`, and runs to the end. The prefix is every message before that point. When K = 0, the suffix is empty and the prefix is the whole conversation.
- Boundary snapping (K > 0): the suffix must start with a `user` message, so an assistant message with tool calls is never separated from its tool results and the summary is always followed by a user turn. If the naive start is not a `user` message, the agent walks back to the nearest earlier `user` message. That message must come after the first non-system message, so the prefix keeps something to compact. If there is none, it walks forward to the nearest later `user` message instead, which keeps fewer messages. If neither exists, nothing is compacted.
- Nothing is compacted if the prefix, after snapping, holds no non-system message.
- Minimum size: when `compaction_min_chars` is positive and the total `content` characters of the prefix's non-system messages is below it, nothing is compacted and no summary call is made. A summary of a tiny history would cost a call and could grow the context.
- Manual compaction: a programmatic caller can compact a session on demand, with the same splitting, snapping and minimum-size rules. It learns whether the history was compacted (true) or skipped (false: nothing to compact, no clean boundary, below the minimum, a failed summary call or an empty summary). Manual compaction does not consult the policy, `compaction_enabled` or the watermark. The `/compact` slash command uses it.
- Compaction runs in two phases, each a separate model call. Both run only after the split, snapping and minimum-size checks pass. Every compaction attempt that reaches the model therefore makes two calls: preservation, then summary.
- Phase 1, preservation: a chat completion to the session's model with `temperature` 0 and `max_tokens` = `compaction_target_tokens`. Its messages are the prefix, as-is, followed by one `user` message with exactly this text (a single line):
  ```
  Context compaction is about to run: the earlier part of this conversation will soon be replaced by a summary. Before that happens, write down everything you will need to keep working on this task afterwards — key facts, decisions made, current state of the work, important file contents or identifiers, and the immediate next steps. Anything you do not record now may be lost. Reply briefly once done.
  ```
  If the reply content, with leading and trailing whitespace removed, is non-empty, two messages are appended to the prefix: that `user` preservation prompt and an `assistant` message whose `content` is the stripped reply. These appended messages only feed the summary call. They are not kept in the conversation afterwards: they are summarized away with the rest of the prefix.
- Phase 2, summary: a chat completion to the session's model with `temperature` 0 and `max_tokens` = `compaction_target_tokens`. Its messages are the prefix (including any phase-1 exchange), followed by one `user` message with exactly this text:
  ```
  Summarize the conversation above into a dense, structured summary optimized for continuing a coding task. Preserve all state needed to keep working without re-reading files.

  Preserve:
  - The current objective and any user constraints
  - Files that have been touched and what changes were made
  - Commands or tools used and their key outcomes
  - Errors, failing tests, or build failures
  - Failed hypotheses or dead ends already explored
  - Unresolved issues or blockers
  - The immediate next step if one was identified

  Style:
  - Plain text with clear sections. Be concise but complete.
  - Summarize outcomes; quote logs only where the exact bytes matter.
  - Report each fact once. Keep uncertainty expressed as uncertainty.
  ```
  The text ends with a newline after the last `Style:` bullet. The prompt states what a good summary looks like; it has no list of things to avoid.
- The summary is the reply content with leading and trailing whitespace removed. On success, the conversation becomes: the system messages from the prefix, then one summary message, then the suffix. The summary message has `role` `assistant`, `content` set to the summary, `compacted_summary: true`, and `ts`, an ISO-8601 timestamp in seconds.
- After a successful compaction the agent:
  - emits a `compaction` event with `summary`, `compacted_turns` (the number of non-system messages in the prefix) and `fmt` (a dim line `[compaction] <n> turns summarized (<chars> chars)`, followed by two spaces and a highlighted `ratio <r>x`, with r to 1 decimal place);
  - appends a `compaction` record to the session trace with `compacted_turns`, `summary_length` (characters), `compaction_ratio` (the ratio rounded to 2 decimal places), `summary` and `ts`;
  - saves the session's message snapshot, so a resumed session starts from the compacted history.
- Compaction ratio = total characters of the `content` of the prefix's non-system messages ÷ characters of the summary. A message with no content counts as 0 characters.
- Context-window rejection: an error from a model call counts as a context-window rejection when either (a) the client already classified it as one, or (b) its HTTP status is 400 or 413 and its message mentions a context or token limit. Case-insensitive phrases that count include "context length", "context window", "token limit", "maximum context", "too many tokens", "prompt is too long", "exceeds the model", "request too large", "input too long" and "reduce the length". The status comes from the error's status code, or from its response's status code. If neither exists, it comes from the message text, in the form `HTTP <nnn>` or `Error code: <nnn>`. A 429 never counts, and neither does an error with no status or a 400 whose message does not mention a limit.
- Overflow recovery in a turn: when a main model call fails with a context-window rejection, the turn does not end with `API error`. Instead the agent:
  1. emits an `error` event with text `Context window exceeded: <error>` and writes an `error` log record with the structured diagnostic fields and `error_kind` set to `context_window_exceeded`;
  2. runs a manual compaction right away, whatever the policy, `compaction_enabled` or the watermark say;
  3. resets the watermark W to 0, so the threshold check can fire again straight away;
  4. if the compaction succeeded and this is the 1st or 2nd consecutive rejection in the turn, emits a `context_overflow_retry` event with `retries` (the consecutive rejection count) and sends the request again with the compacted history;
  5. otherwise (compaction skipped or failed, or a 3rd consecutive rejection) emits a `context_overflow_abort` event with `retries` and ends the turn with the result so far.
  A successful model response resets the consecutive rejection count to 0.
- Summary-call overflow: when the phase-2 summary call itself fails with a context-window rejection, the attempt is not reported as a `Compaction failed` error. Compaction retries with a smaller K. K is halved, rounded down; a K of 1 becomes 0. Before each retry the agent emits a `compaction_retry` event with `keep` (the new K). Each retry repeats the whole attempt (split, snapping, both phases). When the summary call is rejected with K already at 0, compaction gives up and reports "not compacted". The session's configured `compaction_keep_last_turns` does not change.
- Compaction progress: the display line of each per-response `usage` event (`[tokens] …`) includes `compact <p>%`, where p = prompt tokens ÷ the session's `compaction_trigger_tokens` × 100, rounded to a whole number. The parts are separated by `  |  `, and this part sits after the optional cache-write part and before `completion <n>`. It is shown whether or not compaction is enabled. The event's data fields do not change.

## Edge cases
- Compaction disabled → no summary call, whatever the token count.
- `compaction_trigger_tokens` of 0 → the `compact <p>%` part is omitted from the usage line.
- Progress can exceed 100%, for example `compact 104%` on the call that triggers compaction.
- Prompt tokens exactly equal to the threshold trigger compaction (when W + H is below the threshold, as it is at the start). One token below does not.
- With the default threshold of 100000: after a compaction triggered at 100000, later calls reporting 100000 or 100001 do not compact. Only a call reporting more than 125000 compacts again.
- W is updated whenever an attempt is made, even if the attempt compacts nothing (too few messages), fails, or produces an empty summary.
- If the conversation has K or fewer non-system messages, nothing is compacted and no summary call is made.
- If the preservation call fails for any reason, compaction still goes ahead. The agent emits a `compaction_preparation_skipped` event with `reason` (the error text), adds nothing to the prefix, and makes the summary call.
- An empty or whitespace-only preservation reply adds nothing to the prefix and emits no event. The summary call still runs.
- If the summary call fails, the agent emits an `error` event with text `Compaction failed: <error>` and appends a `compaction_error` trace record with `error` (the same `Compaction failed: <error>` text), the structured error diagnostic fields, and `ts`. The conversation stays unchanged and the turn continues.
- If the summary call fails with a rate-limit error (HTTP 429 with no stated retry delay), the `error` event text is `Compaction rate limited: <error>` and the `compaction_error` record also has `error_kind` set to `rate_limit`. Otherwise it is handled like any other failure.
- If the summary is empty or whitespace-only, the conversation stays unchanged and no event is emitted.
- Example of snapping: with K = 2 and messages system, user u1, assistant a1, user u2, assistant with tool calls, tool result, assistant final, the naive start is the tool result. The boundary snaps back to u2, so the result is system, summary, u2, tool-call message, tool result, final.
- Example with no clean boundary: with K = 2 and messages system, user u1, assistant a1, assistant a2, walking back reaches only u1, which is the first non-system message, and there is no later `user` message. Nothing is compacted and no summary call is made.
- A rejection such as `[HTTP 400] … exceeded model token limit: 1048576 (requested: 1680796)` in a plain error message counts as a context-window rejection. `[HTTP 400] … invalid api key` does not: it ends the turn with `API error` and the history is not touched.
- Summary-call overflow with the default K = 2: the attempt at K = 2 is rejected, then the agent retries at K = 1. If that attempt succeeds, the conversation is system, summary, and the last message. That is 4 model calls in total: preservation, rejected summary, preservation, summary.
- A prompt can outgrow the provider's window before the threshold check fires. Examples: the trigger is set above the real window, or one tool result pushes the prompt past the limit between two usage reports. Overflow recovery handles this, so the turn does not fail.
- `every_turn` with too few messages or below the minimum size does nothing at turn end. No summary call is made.
