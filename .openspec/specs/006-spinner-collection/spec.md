---
domain: spinner-collection
version: 3.4.0
status: draft
date: 2026-02-22
---

# Spinner Collection

## Overview

The Spinner Collection subsystem provides the built-in library of named spinner
animations. Spinner data is stored in `yaspin/data/spinners.json` and loaded at
import time via `pkgutil.get_data`. The `Spinners` object exposes each spinner as
a named attribute returning a namedtuple with `frames` and `interval` fields
compatible with the `Spinner` dataclass.

## Philosophy

- Spinner data is data, not code: it lives in a JSON file bundled with the package.
- The `Spinners` object is created dynamically so adding new spinners requires only
  editing `spinners.json` and `constants.py`.
- Custom spinners are always possible by constructing a `Spinner(frames, interval)`
  dataclass directly.

## Requirements

### Requirement 1: Built-in Spinner Library

The package MUST ship `yaspin/data/spinners.json` containing at least 80 named
spinner definitions, each with `frames` (string or list) and `interval` (int, ms).

**Implementation**: yaspin/data/spinners.json, yaspin/constants.py:13 (SPINNER_ATTRS list) — [ADR-0005](../../adr/0005-spinner-data-as-json.md)

#### Scenario: Access named spinner
- GIVEN `from yaspin import Spinners`
- WHEN `Spinners.dots` is accessed
- THEN a namedtuple with `frames` and `interval` is returned

**Tests**: tests/test_spinners.py

---

### Requirement 2: Spinners Object Construction

`Spinners` MUST be constructed by parsing `spinners.json` with a `object_hook`
that converts each JSON object to a `namedtuple` using `collections.namedtuple`.

**Implementation**: yaspin/spinners.py:24 (`_hook`), yaspin/spinners.py:28 (`Spinners`) — [ADR-0005](../../adr/0005-spinner-data-as-json.md)

#### Scenario: JSON to namedtuple
- GIVEN `spinners.json` contains `{"dots": {"frames": "⠋...", "interval": 80}}`
- WHEN `Spinners` is imported
- THEN `Spinners.dots.frames == "⠋..."` and `Spinners.dots.interval == 80`

---

### Requirement 3: Custom Spinner

Callers MUST be able to create a custom spinner by constructing `Spinner(frames, interval)`
directly and passing it to `yaspin()`.

**Implementation**: yaspin/core.py:98 (`Spinner` dataclass), yaspin/__init__.py:4 (export)

#### Scenario: Custom frames
- GIVEN `Spinner(frames="-\\|/", interval=150)`
- WHEN passed to `yaspin(Spinner("-\\|/", 150))`
- THEN the spinner cycles through `-`, `\`, `|`, `/` at 150ms intervals

**Tests**: tests/test_spinners.py

---

### Requirement 4: Lazy Spinner Selection via __getattr__

Callers MAY select a built-in spinner by accessing its name as an attribute on a
`Yaspin` instance. The `__getattr__` implementation MUST lazily import `Spinners`
and set `self.spinner` to the matching entry.

**Implementation**: yaspin/core.py:230 (`__getattr__`), yaspin/core.py:232 (lazy import) — [ADR-0001](../../adr/0001-fluent-interface-via-getattr.md)

#### Scenario: Fluent spinner change
- GIVEN a running `spinner` instance
- WHEN `spinner.dots` is accessed
- THEN `spinner.spinner` is updated to `Spinners.dots` at runtime

**Tests**: tests/test_attrs.py

---

### Requirement 5: Reversal

The spinner MUST support reversing the frame sequence via `reversal=True`, which
reverses the `frames` list/string used for cycling.

**Implementation**: yaspin/core.py:153 (`reversal` parameter), yaspin/core.py:320 (`reversal` setter)

#### Scenario: Reversed animation
- GIVEN `yaspin(reversal=True)`
- WHEN frames are generated
- THEN the sequence is the reverse of the original frames order
