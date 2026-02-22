---
domain: public-api
version: 3.4.0
status: draft
date: 2026-02-22
---

# Public API

## Overview

The Public API subsystem provides the entry points that callers use to create and
use spinners. It lives in `yaspin/api.py` and is re-exported from `yaspin/__init__.py`.
The three public factory functions are `yaspin()`, `kbi_safe_yaspin()`, and
`inject_spinner()`.

## Philosophy

- Keep the API surface minimal: one main factory, one safety wrapper, one injection decorator.
- Callers SHOULD NOT instantiate `Yaspin` directly; they SHOULD use the factory functions.
- `kbi_safe_yaspin` exists as a convenience for the common case of SIGINT handling.

## Requirements

### Requirement 1: Primary Factory Function

`yaspin(*args, **kwargs)` MUST return a `Yaspin` instance constructed with the
provided arguments. It MUST be usable as either a context manager or a decorator.

**Implementation**: yaspin/api.py:22 (`yaspin`)

#### Scenario: Context manager
- GIVEN `with yaspin(text="Working...") as sp:`
- WHEN the block is entered
- THEN `sp` is a live `Yaspin` instance with animation running

#### Scenario: Decorator
- GIVEN `@yaspin(text="Loading...")`
- WHEN the decorated function is called
- THEN the spinner runs for the duration of the function

**Tests**: tests/test_yaspin.py

---

### Requirement 2: KeyboardInterrupt-Safe Factory

`kbi_safe_yaspin(*args, **kwargs)` MUST behave identically to `yaspin()` but
MUST pre-register a `default_handler` for `SIGINT` so the spinner shuts down
cleanly on Ctrl-C.

**Implementation**: yaspin/api.py:121 (`kbi_safe_yaspin`)

#### Scenario: SIGINT handling
- GIVEN `with kbi_safe_yaspin() as sp:`
- WHEN the process receives SIGINT
- THEN `spinner.fail()` and `spinner.stop()` are called and the process exits 0

**Tests**: tests/test_signals.py

---

### Requirement 3: Spinner Injection Decorator

`inject_spinner(*args, **kwargs)` MUST return a decorator that starts a spinner,
passes the live `Yaspin` instance as the **first positional argument** to the
decorated function, and stops the spinner when the function returns.

**Implementation**: yaspin/api.py:96 (`inject_spinner`)

#### Scenario: Spinner injected as first argument
- GIVEN `@inject_spinner(Spinners.dots, text="Processing...")`
  decorates `def process(spinner, data)`
- WHEN `process(my_data)` is called
- THEN `spinner` is a live `Yaspin` instance and `data` is `my_data`

**Tests**: tests/test_inject_spinner.py

---

### Requirement 4: Public Exports

`yaspin/__init__.py` MUST export exactly: `yaspin`, `kbi_safe_yaspin`, `Spinner`,
`inject_spinner` via `__all__`.

**Implementation**: yaspin/__init__.py:3-6
