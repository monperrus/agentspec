# Interruptible nohup_wait and background completion notices

A long `nohup_wait` never freezes the interactive conversation. When the user types a line while the agent is waiting, the wait is cut short so the turn can end and the user's message can run next. The background execution keeps running. When it finishes with nobody waiting on it, the model is told through a `[background]` notice. That notice is either prepended to the next user message or, if the user is idle at the prompt, run as a turn of its own. Library embedders get both halves: a settable interrupt predicate that cuts waits short, and a way to drain unreported completions and render them as the notice.

## Behaviour
- A process-wide wait-interrupt predicate can be set by the host. It answers "is the user waiting to be heard?". By default none is set, and `nohup_wait` behaves exactly as described in the bounded background shell tools spec.
- While waiting, `nohup_wait` checks the predicate on every poll iteration, at least every 100 ms, after checking whether the execution has completed. If the predicate is true, the wait ends at once with the not-completed result, plus these changes:
  - `"interrupted_by": "user_input"`;
  - `hint` becomes `the user typed something while you were waiting, so the wait was cut short — the execution is still running. Stop waiting, end your turn now and answer the user. You will be told automatically when this execution completes; you can also call nohup_wait again later.`;
  - `waited_seconds` is the real time elapsed in this call (e.g. < 1 s even with a 30 m budget);
  - `activity`, `scheduled`/`starts_in_seconds` and `also_completed` are reported as usual.
- The interrupt works both with and without a `howmuch` budget. The execution is never killed by an interrupt.
- The `nohup_wait` tool description ends with this sentence: ` The wait is cut short if the user types something meanwhile (interrupted_by: user_input): end your turn and answer them, the completion will be reported to you automatically.`
- REPL wiring, active only while a turn runs and cleared when the turn ends (also on error or Ctrl-C):
  - Asynchronous (type-ahead) REPL: the predicate is true when at least one typed line is waiting in the type-ahead queue.
  - Synchronous REPL: the predicate is true when standard input is a terminal and has unread input. With non-terminal standard input (e.g. a pipe) it is always false.
- Completion events not consumed by a `nohup_wait` stay queued. A drain operation removes and returns all queued completions without blocking. Each completion is reported by whoever drains it, exactly once. A second drain returns nothing.
- Notice format (model-facing text), for N drained completions:
  - First line: `[background] N nohup execution finished while you were not waiting on it:` when N = 1, or `[background] N nohup executions finished while you were not waiting on them:` when N > 1.
  - Then one line per completion: `- <tool_exec_id>: returncode=<rc> duration=<seconds, 1 decimal>s command=<quoted full started command> stdout=<stdout file> stderr=<stderr file>`. The command is `?` if the execution is unknown.
  - Last line: `Read the output files if you need the details, then continue what this session was doing.`
- Before each user turn, in both REPL variants, pending completions are drained. If there are any, the user's text becomes `<notice>\n\n<user text>`. Slash commands are not affected.
- Idle prompt wake: in both REPL variants, while the main prompt waits for the first keystroke, it checks every 0.25 s for pending completions, but only while standard input has nothing to read. When a completion is pending:
  - the prompt line is erased (carriage return plus clear-to-end-of-line);
  - the completions are drained and the notice is printed dimmed;
  - the notice runs as a turn, in the same mode (sync or async) as that REPL;
  - then the prompt is shown again.

## Edge cases
- A line the user has started typing is never taken over: the wake check only runs while standard input is empty, and after the first keystroke the prompt reads normally.
- If the predicate is true when `nohup_wait` starts and the execution has already completed, the completed result is returned (completion is checked first).
- If a wake fires but the completions were already consumed (e.g. by a `nohup_wait`), no turn runs and the prompt is shown again.
- In the asynchronous REPL, the typed line that caused the interrupt runs as the next queued turn after the interrupted turn ends. In the synchronous REPL, it is read by the next prompt.
