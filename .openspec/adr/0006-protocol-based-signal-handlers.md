# ADR-0006: Protocol-Based Signal Handler Type System

**Status**: Accepted

**Date**: 2026-02-22 (reverse-engineered from codebase)

**Decision ID**: 0006-protocol-based-signal-handlers

---

## Context

The standard `signal.signal()` API requires handlers with signature `(signum, frame)`.
Yaspin's spinner needs to be accessible inside signal handlers so they can call
`stop()` and `fail()`. Passing the spinner via closure is one option, but it creates
coupling between the handler and the specific spinner instance.

## Decision

A `SignalHandlerProtocol` (a `typing.Protocol` with `@runtime_checkable`) is defined
requiring signature `(signum: int, frame: Any, spinner: Yaspin) -> None`.

At registration time (`_register_signal_handlers`), if the handler is `callable` and
an instance of `SignalHandlerProtocol`, it is wrapped with `functools.partial(handler,
spinner=self)` to produce a 2-argument callable acceptable by `signal.signal`.

Two pre-built handlers (`default_handler`, `fancy_handler`) follow this protocol.

## Consequences

### Positive
- Signal handlers can be defined independently of any specific spinner instance
- `functools.partial` injection is transparent to the handler; it just receives the
  spinner as a keyword argument
- `@runtime_checkable` allows `isinstance` checks to distinguish protocol-conforming
  handlers from `signal.SIG_DFL`/`signal.SIG_IGN` integers
- Built-in handlers serve as self-documenting examples of the pattern

### Negative
- Protocol conformance is checked structurally at runtime, not statically; a handler
  with wrong argument names passes `isinstance` but fails when called
- The 3-argument requirement (vs standard 2-argument) diverges from Python's signal
  module convention, which may surprise contributors

### Neutral
- `signal.SIG_DFL` and `signal.SIG_IGN` are valid values in `sigmap` and are passed
  directly to `signal.signal` without wrapping
