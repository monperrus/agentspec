# Machine-readable tool failure flag

A tool result's text is written for the model, so a failure used to be recognisable only by its leading `ERROR: ` sentence. Failed tool calls now also mark their metadata object with `ok: false` and an `error` message, so code that acts on the outcome (a protocol host mapping it to an `isError` flag, a hook, a dashboard) can test the metadata instead of pattern-matching English text. The text the model sees is unchanged.

## Behaviour
- A tool call returns a text result and a metadata object. On failure, the metadata contains `ok: false` and `error: <message>`, alongside `result`.
- On success, the metadata has no `ok` key and no `error` key. Consumers treat a missing `ok` as success, i.e. success ⇔ `ok` is absent or true.
- Failures that set the flag, where `error` holds the same text as the model-facing result (and as `result`):
  - dispatch failures: a dispatch entry whose implementation cannot be resolved, arguments that don't match the implementation (wrong names/values), and any error escaping the implementation (internal tool failure);
  - built-in tool internal errors (`ERROR: <tool> internal error: ...`), e.g. reading a missing file, where `error` mentions the error type name;
  - refusals the built-in tools detect themselves: string-replace old text not found, a patch without an `*** Update File:` line, a missing file path, a path that is not an existing directory, a missing `command`, an ask-the-user call with no question or with no user input available.
- The search tool keeps returning its JSON `{"error": "<message>"}` text on failure (timeout after 30 s, or failure to start the search); its metadata gets `ok: false` and `error: <message>` (the bare message, not the JSON).

## Edge cases
- A successful result whose text happens to start with `ERROR:` (e.g. reading a file whose first line is `ERROR: ...`) is a success: `ok` is absent.
- Built-in tools that previously reported failure with metadata `result: "error"` now carry the failure message in `result` (same as the model-facing text) plus `ok: false` and `error`.
