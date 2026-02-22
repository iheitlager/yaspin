---
domain: stream-handling
version: 3.4.0
status: draft
date: 2026-02-22
---

# Stream Handling

## Overview

The Stream Handling subsystem manages all output to the terminal. It wraps the
target `TextIO` stream in a `SafeStreamWrapper` that defends against writes to
closed streams and provides TTY detection used by the color and cursor systems.

## Philosophy

- The spinner SHOULD default to `sys.stdout` but MUST accept any `TextIO` stream.
- Closed-stream writes MUST be silently ignored by default (to avoid crashes during
  teardown), with optional warning emission for debugging.
- TTY detection MUST be delegated to the stream's `isatty()` method so that
  behavior is correct for `sys.stderr`, `StringIO`, pipes, and real terminals alike.

## Requirements

### Requirement 1: Default Stream

When no `stream` argument is provided, the spinner MUST write to `sys.stdout`.

**Implementation**: yaspin/core.py:162 (stream initialization in `__init__`)

#### Scenario: Default output
- GIVEN `yaspin()` with no stream argument
- WHEN the spinner runs
- THEN frames are written to `sys.stdout`

---

### Requirement 2: Custom Stream Support

The spinner MUST accept any `TextIO`-compatible stream via the `stream` constructor
argument, including `sys.stderr`, file objects, and `StringIO`.

**Implementation**: yaspin/core.py:158 (`stream` parameter), yaspin/core.py:162

#### Scenario: stderr stream
- GIVEN `yaspin(stream=sys.stderr)`
- WHEN the spinner runs
- THEN frames are written to `sys.stderr`

**Tests**: tests/test_stream.py

---

### Requirement 3: Closed Stream Resilience

`SafeStreamWrapper` MUST silently discard writes and flushes to closed streams.
If `warn_on_closed_stream=True`, it MUST emit a `UserWarning` on the **first**
write attempt to a closed stream and suppress subsequent warnings.

**Implementation**: yaspin/core.py:51 (`SafeStreamWrapper`), yaspin/core.py:59 (`write`),
yaspin/core.py:72 (`flush`)

#### Scenario: Write to closed stream (silent)
- GIVEN `warn_on_closed_stream=False` (default)
- WHEN the stream is closed before the spinner stops
- THEN writes are silently discarded and no exception is raised

#### Scenario: Write to closed stream (warn once)
- GIVEN `warn_on_closed_stream=True`
- WHEN the stream closes and the spinner tries to write
- THEN exactly one `UserWarning` is emitted and subsequent writes are silent

**Tests**: tests/test_stream.py

---

### Requirement 4: TTY Detection

`SafeStreamWrapper.isatty()` MUST return `False` for closed streams and delegate
to the underlying stream's `isatty()` otherwise.

**Implementation**: yaspin/core.py:78 (`SafeStreamWrapper.isatty`)

#### Scenario: TTY stream
- GIVEN a real TTY stream
- WHEN `isatty()` is called
- THEN `True` is returned and ANSI codes and cursor sequences are enabled

#### Scenario: Non-TTY stream
- GIVEN a `StringIO()` or pipe
- WHEN `isatty()` is called
- THEN `False` is returned and ANSI codes are suppressed

**Tests**: tests/test_stream.py, tests/test_in_out.py

---

### Requirement 5: Pipe/Non-TTY Line Clearing

When the stream is not a TTY, the spinner MUST clear the current line by writing
spaces equal to the tracked line length rather than ANSI EL (`\r\033[K`).

**Implementation**: yaspin/core.py:716 (`_clear_line`)

#### Scenario: Pipe output
- GIVEN a pipe or non-TTY stream
- WHEN `_clear_line()` is called
- THEN `\r` followed by spaces of the current line length is written instead
  of the ANSI escape sequence

**Tests**: tests/test_pipes.py
