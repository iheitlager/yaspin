---
domain: signal-handling
version: 3.4.0
status: draft
date: 2026-02-22
---

# Signal Handling

## Overview

The Signal Handling subsystem allows callers to register custom POSIX signal
handlers that receive the live `Yaspin` instance, enabling clean spinner teardown
on interrupts. Handlers are installed on `start()` and restored to defaults on
`stop()`. Two pre-built handlers are provided: `default_handler` and `fancy_handler`.

## Philosophy

- Signal handlers MUST receive the spinner instance so they can call `stop()`/`fail()`.
- The `SignalHandlerProtocol` Protocol enforces the correct 3-argument signature
  (`signum`, `frame`, `spinner`).
- SIGKILL registration MUST be rejected (cannot be caught on POSIX).
- Default signal handlers MUST be restored on stop to avoid leaking custom handlers.

## Requirements

### Requirement 1: Signal Map Registration

The spinner MUST accept a `sigmap` dict mapping `signal.Signals` to handler callables
or `signal.SIG_DFL`/`signal.SIG_IGN`. Handlers MUST be installed on `start()`.

**Implementation**: yaspin/core.py:195 (`_sigmap`), yaspin/core.py:353 (registration on start),
yaspin/core.py:660 (`_register_signal_handlers`)

#### Scenario: Custom SIGTERM handler
- GIVEN `sigmap={signal.SIGTERM: my_handler}`
- WHEN the process receives SIGTERM
- THEN `my_handler(signum, frame, spinner=sp)` is called

**Tests**: tests/test_signals.py

---

### Requirement 2: SIGKILL Rejection

The spinner MUST raise `ValueError` if `signal.SIGKILL` appears in `sigmap`.

**Implementation**: yaspin/core.py:675 (SIGKILL check in `_register_signal_handlers`)

#### Scenario: SIGKILL in sigmap
- GIVEN `sigmap={signal.SIGKILL: my_handler}`
- WHEN `start()` is called
- THEN a `ValueError` is raised with an explanatory message

**Tests**: tests/test_signals.py

---

### Requirement 3: Handler Restoration on Stop

All custom signal handlers installed via `sigmap` MUST be restored to their
pre-spinner defaults when `stop()` is called.

**Implementation**: yaspin/core.py:383 (reset on stop), yaspin/core.py:699 (`_reset_signal_handlers`)

#### Scenario: Handler restoration
- GIVEN a spinner with a SIGINT handler
- WHEN `stop()` is called
- THEN the original SIGINT handler is restored

**Tests**: tests/test_signals.py

---

### Requirement 4: SignalHandlerProtocol

Custom signal handlers provided via `sigmap` that are callable MUST conform to
`SignalHandlerProtocol`: they MUST accept `(signum: int, frame: Any, spinner: Yaspin)`.
The spinner instance MUST be injected via `functools.partial` at registration time.

**Implementation**: yaspin/core.py:107 (`SignalHandlerProtocol`), yaspin/core.py:689 (partial injection)

#### Scenario: Protocol-conforming handler
- GIVEN a handler `def my_handler(signum, frame, spinner):`
- WHEN installed via `sigmap` and signal fires
- THEN `spinner` receives the live `Yaspin` instance

---

### Requirement 5: Built-in Signal Handlers

Two built-in handlers MUST be provided:
- `default_handler`: calls `spinner.fail()`, `spinner.stop()`, exits 0
- `fancy_handler`: calls `spinner.red.fail("✘")`, `spinner.stop()`, exits 0

**Implementation**: yaspin/core.py:112 (`default_handler`), yaspin/core.py:124 (`fancy_handler`)

#### Scenario: kbi_safe_yaspin uses default_handler
- GIVEN `kbi_safe_yaspin()`
- WHEN Ctrl-C is pressed
- THEN `default_handler` fires, spinner fails gracefully, process exits 0

**Tests**: tests/test_signals.py
