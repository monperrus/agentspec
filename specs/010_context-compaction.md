# Context compaction

Long sessions compact automatically. When a model call reports that its prompt reached a configurable token threshold, the agent asks the model to summarize the older part of the conversation into a summary aimed at continuing the task. That summary replaces the older messages. The most recent messages stay as they were. Compaction is on by default.

## Behaviour
- Configuration comes from these agent spec fields. A programmatic caller can override each one per session, for single tasks, the sync REPL and the async REPL. An explicit override wins over the spec field, and the spec field wins over the default:
  - `compaction_enabled` (default `true`).
  - `compaction_trigger_tokens` (default `100000`): the prompt-token threshold.
  - `compaction_target_tokens` (default `20000`): the output-token limit (`max_tokens`) for the summary call.
  - `compaction_keep_last_turns` (default `2`): how many of the most recent non-system messages stay unchanged.
- Triggering relies only on usage metadata from the server. The agent never counts tokens locally. After each main model call's usage has been recorded, compaction runs when it is enabled and the response's `usage.prompt_tokens` is ≥ `compaction_trigger_tokens` and also > W + H (see hysteresis below). A missing value counts as 0. This check comes before the session token-budget check.
- Hysteresis: the session keeps a watermark W, which starts at 0. Each time compaction is attempted, W becomes the prompt-token count that triggered the attempt, which is the count before compaction. H = `compaction_trigger_tokens` ÷ 4, rounded down. Compaction can therefore fire again only once the prompt count has grown by more than a quarter of the threshold past the last triggering count. This stops a "compaction storm": the remaining context (system, summary and kept messages) can still sit at or above the threshold, and without the watermark every following turn would compact again.
- `/clear` resets W to 0.
- Split: the kept suffix begins at the K-th most recent non-system message, where K = `compaction_keep_last_turns`, and runs to the end. The prefix is every message before that point. When K = 0, the suffix is empty and the prefix is the whole conversation.
- The summary call is a separate chat completion to the session's model with `temperature` 0 and `max_tokens` = `compaction_target_tokens`. Its messages are the prefix, as-is, followed by one `user` message with exactly this text:
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

  Avoid:
  - Conversational filler or chatter
  - Repeated log output (summarize outcomes, don't quote logs verbatim)
  - Redundant observations
  - Rewriting uncertainty as certainty

  Format the summary as plain text with clear sections. Be concise but complete.
  ```
- The summary is the reply content with leading and trailing whitespace removed. On success, the conversation becomes: the system messages from the prefix, then one summary message, then the suffix. The summary message has `role` `assistant`, `content` set to the summary, `compacted_summary: true`, and `ts`, an ISO-8601 timestamp in seconds.
- After a successful compaction the agent:
  - emits a `compaction` event with `summary`, `compacted_turns` (the number of non-system messages in the prefix) and `fmt` (a dim line `[compaction] <n> turns summarized (<chars> chars)`, followed by two spaces and a highlighted `ratio <r>x`, with r to 1 decimal place);
  - appends a `compaction` record to the session trace with `compacted_turns`, `summary_length` (characters), `compaction_ratio` (the ratio rounded to 2 decimal places), `summary` and `ts`;
  - saves the session's message snapshot, so a resumed session starts from the compacted history.
- Compaction ratio = total characters of the `content` of the prefix's non-system messages ÷ characters of the summary. A message with no content counts as 0 characters.
- Compaction progress: the display line of each per-response `usage` event (`[tokens] …`) includes `compact <p>%`, where p = prompt tokens ÷ the session's `compaction_trigger_tokens` × 100, rounded to a whole number. The parts are separated by `  |  `, and this part sits after the optional cache-write part and before `completion <n>`. It is shown whether or not compaction is enabled. The event's data fields do not change.

## Edge cases
- Compaction disabled → no summary call, whatever the token count.
- `compaction_trigger_tokens` of 0 → the `compact <p>%` part is omitted from the usage line.
- Progress can exceed 100%, for example `compact 104%` on the call that triggers compaction.
- Prompt tokens exactly equal to the threshold trigger compaction (when W + H is below the threshold, as it is at the start). One token below does not.
- With the default threshold of 100000: after a compaction triggered at 100000, later calls reporting 100000 or 100001 do not compact. Only a call reporting more than 125000 compacts again.
- W is updated whenever an attempt is made, even if the attempt compacts nothing (too few messages), fails, or produces an empty summary.
- If the conversation has K or fewer non-system messages, nothing is compacted and no summary call is made.
- If the summary call fails, the agent emits an `error` event with text `Compaction failed: <error>` and appends a `compaction_error` trace record with `error` and `ts`. The conversation stays unchanged and the turn continues.
- If the summary is empty or whitespace-only, the conversation stays unchanged and no event is emitted.
- The kept suffix is cut by message count, not by user/assistant pairs. It can therefore begin in the middle of an exchange, for example with a tool result.
