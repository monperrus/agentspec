# Programmatic CLI invocation

A launcher running in the same process as the agent can start the command-line front end with an explicit argument list, composed as data, instead of rewriting the process's own command-line arguments. The process arguments stay untouched, so nothing else in the process sees a modified command line and the launcher does not depend on inserting arguments at fixed positions.

## Behaviour
- The CLI entry point and its argument parser both accept an optional argument list. The list excludes the program name: its first element is the first real argument (e.g. the model id).
- A given list is parsed exactly as the same words typed on the command line would be: the positional model, the positional task words, and every flag (e.g. `["my-model", "do", "the", "thing"]` yields model `my-model` and task `do the thing`; `["m", "--context-window", "4096", "--non-interactive"]` yields a 4096-token context window and non-interactive mode).
- With no list given, the process's command-line arguments (without the program name) are used, as before.
- Invoking the entry point with an explicit list never reads or modifies the process's command-line arguments; they are identical before and after the call.
- Validation and startup behave as for a normal command-line launch: an invalid value in the given list is rejected with the same error and exit as when typed (e.g. `--context-window 0` exits with an error naming `--context-window`), and terminal hyperlink rewriting is still enabled before parsing when standard output is a TTY.

## Edge cases
- An empty list is an explicit list, not "no list": it is parsed as a command line with no arguments (so the missing model is an argument error), and the process arguments are not consulted.
