# Type-ahead input queue

In the interactive REPL and the interactive CLI loop, the user can keep typing while a turn is running (for example while tools execute). Lines entered during a turn are queued and handled after the turn finishes, as slash commands or as follow-on turns.

## Behaviour
- While a turn runs, the agent keeps reading lines from standard input in the background without blocking the turn.
- After standard input has been idle for 0.5 s during a turn, a dim `> ` prompt is printed with no leading newline, so the user knows input is accepted. It is printed at most once per idle period. After a line is read, it shows again once input has been idle for another 0.5 s.
- Each non-blank line read during a turn is queued, and the agent prints a dim confirmation `[queued — will run after current turn]` on its own line. The confirmation starts with a carriage return so it overwrites the `> ` prompt. Blank or whitespace-only lines are ignored and not confirmed.
- When the turn ends, the conversation snapshot is saved, and then the queued lines are processed in the order they were typed:
  - A line that is an exit word (`exit`, `quit`, `q`, case-insensitive, ignoring surrounding whitespace) is dropped. It does not end the session.
  - A slash command runs at once, and the snapshot is saved after it.
  - Any other line is added to a pending list of instructions, keeping its text as typed.
- Pending instructions then run one at a time as ordinary turns. Lines typed during each follow-on turn are collected and processed in the same way, so queuing nests to any depth. Control returns to the normal prompt when no instructions are pending.
- Error handling for follow-on turns is the same as for a normal turn: an unexpected error fires an `error` event, writes an `error` log record, and processing continues with the next pending instruction.

## Edge cases
- If a turn is interrupted with Ctrl-C, the rest of the pending instructions and any lines queued during that turn are discarded, and the agent returns to the normal prompt.
- Slash commands queued during a turn run before any instruction queued in the same batch, because the whole batch is scanned (running slash commands) before any queued instruction runs.
- End of input on standard input stops the background reading for that turn. Nothing else is queued.
- Background reading stops when the turn ends. A line typed after that point is read by the normal prompt, not the queue.
