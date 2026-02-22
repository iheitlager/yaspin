# ADR-0001: Fluent Interface via `__getattr__`

**Status**: Accepted

**Date**: 2026-02-22 (reverse-engineered from codebase)

**Decision ID**: 0001-fluent-interface-via-getattr

---

## Context

Yaspin needs to let callers configure spinner appearance (color, side, built-in
spinner name, text attributes) without requiring verbose keyword-argument syntax
on every use. A common pattern in Python libraries is to allow dot-chained calls
that feel like CSS class names.

## Decision

`Yaspin.__getattr__` intercepts attribute access for names that match:
- Built-in spinner names (from `SPINNER_ATTRS` in `constants.py`)
- termcolor `COLORS`, `HIGHLIGHTS`, and `ATTRIBUTES` keys
- The strings `"left"` and `"right"`

When a match is found, the appropriate property setter is called and `self` is
returned, enabling chaining like `spinner.cyan.bold.right.dots`.

Unrecognized names raise `AttributeError` with a helpful message.

## Consequences

### Positive
- Extremely concise caller-side syntax: `yaspin().cyan.bold.dots`
- Requires no changes to add new named spinners (just update `spinners.json` and `constants.py`)
- Discoverable in REPL via tab-completion (attributes exist in data structures)

### Negative
- Lazy attribute resolution can be surprising: typos raise `AttributeError` at
  runtime rather than being caught statically
- Cyclomatic complexity of `__getattr__` is elevated (7) due to the multiple dispatch branches
- Type checkers cannot verify chained attribute usage without custom stubs

### Neutral
- All legitimate dynamic attributes still route through typed property setters,
  so validation and side effects are preserved

## Related Specs

- [001-spinner-engine](../specs/001-spinner-engine/spec.md) — Req 9: Fluent Interface
- [003-color-system](../specs/003-color-system/spec.md) — Req 5: Color Function Recomposition (chaining triggers recompose)
- [006-spinner-collection](../specs/006-spinner-collection/spec.md) — Req 4: Lazy Spinner Selection via `__getattr__`
