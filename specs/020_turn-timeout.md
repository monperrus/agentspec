# Cooperative turn timeout

A library caller running a single agent turn can give it a time limit in seconds. When the limit expires, the turn is cancelled cooperatively: nothing in flight is interrupted, and the turn fails with a timeout error the next time it reaches a cancellation boundary.

## Behaviour
- The time limit is optional; without it a turn has no time limit.
- The timer starts when the turn starts and is stopped when the turn ends, whether it succeeds or fails. A timer never outlives its turn.
- When the limit expires, the turn is marked cancelled. A caller-supplied cancellation token, if any, is the one that gets cancelled, so the caller can see the cancellation too.
- Cancellation boundaries are before each model call and right after each model call returns successfully. At a boundary, a cancelled turn fails with:
  - a timeout error whose message says the turn timed out, when the time limit caused the cancellation;
  - the ordinary user-interrupt abort, when the cancellation came from anything else (Ctrl-C or another thread).
- An in-flight model request or tool call is not interrupted. Its response is discarded if the limit expired while it ran: the turn does not process the reply, and it produces no final answer from it.

## Edge cases
- A negative limit is rejected before the turn starts, with an error saying the timeout must be non-negative or none.
- A limit of 0 is allowed; the turn times out at its first boundary once the timer has fired.
- If the caller cancels explicitly and the time limit has not expired, the turn reports an interrupt, not a timeout.
