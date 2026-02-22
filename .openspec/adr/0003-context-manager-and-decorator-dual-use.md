# ADR-0003: Context Manager and Decorator Dual-Use

**Status**: Accepted

**Date**: 2026-02-22 (reverse-engineered from codebase)

**Decision ID**: 0003-context-manager-and-decorator-dual-use

---

## Context

Users want to display a spinner around both ad-hoc code blocks (`with` statement)
and reusable functions (`@decorator`). Providing two separate APIs would fragment
the library surface; a single object that works in both modes is preferred.

## Decision

`Yaspin` implements both `__enter__`/`__exit__` (context manager protocol) and
`__call__` (callable/decorator protocol). When called with a function argument,
`__call__` wraps the function in a `with self:` block using `functools.wraps`.

The factory functions `yaspin()` and `kbi_safe_yaspin()` return a `Yaspin` instance,
so both styles use the same API: `with yaspin():` or `@yaspin()`.

`inject_spinner()` in `api.py` adds a third pattern for passing the spinner as a
function argument.

## Consequences

### Positive
- Single unified API; callers choose their preferred style
- `functools.wraps` preserves docstrings and metadata on decorated functions
- The same `Yaspin` instance (and thus the same state) is used regardless of usage style

### Negative
- `__call__` makes `Yaspin` both a context manager and a higher-order function,
  which may be unexpected to readers unfamiliar with this pattern
- `@yaspin()` must be called (with parens) even without arguments, because `yaspin`
  returns an instance, not a class

### Neutral
- All three patterns (`with`, `@`, `inject_spinner`) ultimately call `start()`
  and `stop()`, making the lifecycle code a single authority

## Related Specs

- [001-spinner-engine](../specs/001-spinner-engine/spec.md) — Req 1: Lifecycle Management (context manager and decorator entry points)
- [002-public-api](../specs/002-public-api/spec.md) — Req 1: Primary Factory Function
