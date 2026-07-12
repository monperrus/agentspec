# Clickable terminal hyperlinks

When standard output is an interactive terminal, the command-line agent rewrites every bare `http://` or `https://` URL it prints into an OSC 8 terminal hyperlink, so the URL is clickable in supporting terminals. A host program embedding the agent as a library can turn on the same rewriting explicitly.

## Behaviour
- The command-line entry point enables rewriting at startup, before parsing arguments, if and only if standard output is a TTY.
- A host program can enable rewriting with a single library call that takes no arguments and returns nothing; it applies to all subsequent writes to standard output from any part of the process.
- A URL is a maximal run starting with `http://` or `https://` and continuing up to (not including) the next whitespace character.
- Each URL is replaced by `ESC ] 8 ; ; <url> ESC \ <url> ESC ] 8 ; ; ESC \`, i.e. the visible text is the URL itself and the link target is the same URL.
- Rewriting is line-buffered: written text is held until a newline arrives, then each complete line is rewritten and emitted with its newline. A flush emits any pending partial line (rewritten) and then flushes the underlying output.
- Text without URLs passes through byte-for-byte unchanged; all other output-stream properties and operations behave as for the original standard output.

## Edge cases
- When standard output is not a TTY (piped, redirected), the command-line agent does not rewrite anything.
- Enabling more than once is a no-op; URLs are never wrapped twice by repeated enabling.
- Trailing punctuation, quotes or escape sequences directly attached to a URL (e.g. `https://x.org).`) are included in both link target and visible text, since only whitespace ends a URL.
- Output written without a trailing newline and never flushed stays buffered and does not appear.
- Standard error is never rewritten.
