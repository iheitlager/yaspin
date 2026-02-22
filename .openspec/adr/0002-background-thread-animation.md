# ADR-0002: Background Thread Animation Model

**Status**: Accepted

**Date**: 2026-02-22 (reverse-engineered from codebase)

**Decision ID**: 0002-background-thread-animation

---

## Context

A terminal spinner must update the display at regular intervals (typically 80–200ms)
while the caller's long-running operation proceeds concurrently. The animation loop
and the application work must not block each other.

## Decision

Yaspin spawns a `threading.Thread` that runs `_spin()` in a loop, sleeping for
`_interval` milliseconds between frames via `threading.Event.wait()`. The thread is
started in `start()` and terminated by setting a `threading.Event` (`_stop_spin`)
in `stop()`, followed by `thread.join()`.

A `threading.Lock` (`_stream_lock`) guards all writes to the output stream to
prevent interleaving between the spinner thread and any main-thread writes (via
`write()`, `ok()`, or `fail()`).

## Consequences

### Positive
- Non-blocking: caller code runs concurrently with animation
- Clean termination: `Event`-based stop is cooperative and avoids daemon thread risks
- Thread-safe output: `_stream_lock` prevents corrupted terminal state
- `Event.wait(timeout)` is used instead of `time.sleep()` so the stop signal is
  detected promptly without busy-waiting

### Negative
- GIL means threads share one CPU core in CPython; acceptable since the spinner is
  I/O-bound
- Context manager `__exit__` must check `is_alive()` to avoid double-stop
- Signal handlers must use `functools.partial` to inject the spinner because
  `signal.signal` only accepts 2-argument callables

### Neutral
- The `threading` approach is simpler than `asyncio`; async support is not provided
  but the lock pattern is compatible with async-bridging if added later
