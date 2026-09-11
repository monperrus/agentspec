# Environment awareness in the system prompt

At session start the agent appends an environment block to the system prompt. It tells the model who the user is, the state of the git repository, where it runs, on which platform, the current time, where to put temporary files and which model and harness it is.

## Behaviour
- The block is appended last, after the caller supplement, the global instructions file and `AGENTS.md`, separated from the preceding text by a blank line. It is added to every newly created session, in all call-delivery modes.
- The block is a sequence of lines joined by newlines, in this order:
  1. `## Environment`
  2. `User: <identity>` (see below), omitted when no identity is known.
  3. The git status block (see below), omitted outside a git work tree.
  4. `Working directory: <absolute current working directory>`
  5. `OS: <system name> <kernel release> (<machine architecture>)`, e.g. `OS: Linux 6.8.0 (x86_64)`.
  6. `Current date/time: <YYYY-MM-DD HH:MM:SS> (<timezone name> timezone)`, in local time. If the timezone has no name, `local` is used instead.
  7. `Scratchpad (for temporary files): <path>`: a per-working-directory directory directly under the system temporary directory (see below). The agent creates it (with parents) if it is missing.
  8. `Model: <model>`, followed by ` (version <version>)` when the spec has a non-empty `version` field.
  9. `Harness: <harness name> <harness version>`: the agent's own fixed product name followed by its installed release version string (the same version it reports elsewhere). Always present.
- Identity: `unix user: <login name>` when the login name can be determined. Then, when git `user.name` or `user.email` is configured (as seen from the working directory), `git identity: <name> <<email>>`, with the absent part left out (`git identity: <name>` or `git identity: <<email>>`). The two parts are joined by `; `. If only the git part exists, it appears alone.
- Git status block, when the working directory is inside a git work tree:
  - `Git: on branch <current branch>`, or `(unknown)` in place of the branch when it cannot be determined. A detached HEAD shows as `HEAD`.
  - `  last commit: <subject of the latest commit>` when there is one.
  - If the working tree has changes: `  changed files (<n>):`, then up to the first 20 short-format (porcelain) status lines, each indented by four spaces. If there are more than 20, one more line `    … (<n − 20> more)` follows.
  - Otherwise: `  working tree clean`.
- Scratchpad directory name: `<fixed agent-specific prefix>-<basename of the working directory>-<hash>`, where `<hash>` is the first 8 lowercase hex characters of the SHA-1 digest of the absolute working-directory path (as a UTF-8 string).

## Edge cases
- Git missing, not a repository, or any git query failing to start or taking more than 5 seconds: the git status block is omitted and the git identity part is left out. The session still starts.
- A repository with no commits yet: the `last commit` line is omitted.
- Unset or empty git config values count as absent.
- Two working directories with the same basename but different paths get different scratchpads. Sessions started in the same working directory always get the same scratchpad, so it is stable across sessions of one project.
- The block is computed once, at session creation. It is not refreshed when the directory, branch or time changes later in the session. A session restored in memory keeps its stored system prompt unchanged.
