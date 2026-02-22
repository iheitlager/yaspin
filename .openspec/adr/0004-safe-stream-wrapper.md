# ADR-0004: SafeStreamWrapper for Closed-Stream Resilience

**Status**: Accepted

**Date**: 2026-02-22 (reverse-engineered from codebase)

**Decision ID**: 0004-safe-stream-wrapper

---

## Context

In long-running programs, test harnesses, or Jupyter environments, the stream
passed to a spinner can be closed before the spinner's `stop()` is called. Without
protection, writes to a closed stream raise `ValueError: I/O operation on closed
file`, crashing the process during cleanup.

## Decision

All stream I/O is routed through `SafeStreamWrapper`, which:
1. Checks `stream.closed` before every `write()` and `flush()` call
2. Silently discards the operation if the stream is closed
3. Optionally emits a single `UserWarning` (suppressed on repeats) if
   `warn_on_closed=True`
4. Delegates `isatty()` and other attributes to the underlying stream via
   `__getattr__` pass-through

## Consequences

### Positive
- Crashes from closed-stream writes are eliminated by default
- Optional warning mode makes debugging stream lifecycle issues easier
- Single-warning pattern avoids spamming the warning log
- `__getattr__` delegation keeps the wrapper transparent for other stream attributes

### Negative
- An extra abstraction layer over every stream write adds marginal overhead
- Silent discard may obscure bugs where a stream is accidentally closed early
  (mitigated by `warn_on_closed_stream=True`)

### Neutral
- `SafeStreamWrapper` is internal (not exported); callers always pass a raw `TextIO`

## Related Specs

- [004-stream-handling](../specs/004-stream-handling/spec.md) — Req 3: Closed Stream Resilience, Req 4: TTY Detection
