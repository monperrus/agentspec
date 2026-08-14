# Multiline paste input

At the interactive REPL prompt, a multiline clipboard paste is submitted as one task, not split into one turn per line. The prompt keeps reading lines while more input arrives in quick succession and stops once input has been idle briefly.

## Behaviour
- The REPL prints its prompt, then reads one line from standard input. Terminal non-printing markers in the prompt (the `\x01` and `\x02` bytes that bracket escape sequences for line editors) are stripped before printing.
- After the first line, the REPL keeps reading further lines as long as another line becomes available on standard input within a short idle timeout. All lines read this way form one task, joined with `\n`.
- Every pasted line is included in the task. Line-editor read-ahead must not hide buffered lines from the idle check. Reading lines directly from standard input, not through a line-editing input call that may buffer ahead, meets this requirement.
- Trailing newlines are removed from each line and from the whole task.
- A non-empty task is added to the prompt history as a single entry containing the full multiline text. An empty task is not added.

## Edge cases
- End of input before any line is read ends the REPL, exactly as end of input at the prompt does.
- End of input after the first line stops the draining. The lines read so far are submitted as the task.
- A single typed line followed by no further input within the idle timeout is submitted on its own, as usual.
