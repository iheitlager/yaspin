---
domain: spinner-engine
version: 3.4.0
status: draft
date: 2026-02-22
---

# Spinner Engine

## Overview

The Spinner Engine is the core subsystem of yaspin. It manages the lifecycle of a
terminal spinner: initialization, animation in a background thread, visibility
control, output composition, and graceful teardown. The central class is `Yaspin`
in `yaspin/core.py`.

## Philosophy

- A spinner runs in a **background thread** so it never blocks the caller's work.
- All terminal writes are protected by a **stream lock** to prevent interleaving.
- The spinner is usable as both a **context manager** and a **decorator**, enabling
  the same API surface for both styles.
- A **fluent interface** via `__getattr__` lets callers chain display attributes
  (`spinner.cyan.bold.right`) before or during use.

## Requirements

### Requirement 1: Lifecycle Management

The `Yaspin` class MUST implement a start/stop lifecycle that spawns a background
thread for animation and terminates it cleanly on stop.

**Implementation**: yaspin/core.py:340 (`start`), yaspin/core.py:370 (`stop`)

#### Scenario: Context manager usage
- GIVEN a caller enters a `with yaspin()` block
- WHEN `__enter__` is called
- THEN the spinner MUST start (`start()` called), the thread is alive, and
  the spinner instance is returned

#### Scenario: Decorator usage
- GIVEN `@yaspin(text="Loading...")` decorates a function
- WHEN the decorated function is called
- THEN the spinner MUST start before the function body executes and stop
  after it returns (or raises)

**Tests**: tests/test_yaspin.py, tests/test_in_out.py

---

### Requirement 2: Background Thread Animation

The spinner MUST animate in a dedicated `threading.Thread`, cycling through frames
at the configured interval without blocking the main thread.

**Implementation**: yaspin/core.py:540 (`_spin`), yaspin/core.py:360 (thread creation)

#### Scenario: Non-blocking animation
- GIVEN a spinner is started
- WHEN the main thread performs work inside the context
- THEN both execute concurrently and the terminal shows animated frames

#### Scenario: Thread teardown
- GIVEN the spinner is running
- WHEN `stop()` is called
- THEN `_stop_spin` event is set, the thread joins, and no more frames are written

**Tests**: tests/test_yaspin.py, tests/test_in_out.py

---

### Requirement 3: Thread-Safe Stream Writes

All terminal writes MUST be protected by a `threading.Lock` (`_stream_lock`) to
prevent interleaving between the spinner thread and the main thread.

**Implementation**: yaspin/core.py:163 (`_stream_lock`), yaspin/core.py:564 (lock in `_spin`),
yaspin/core.py:477 (lock in `write`)

#### Scenario: Concurrent write
- GIVEN a spinner is running
- WHEN `spinner.write("message")` is called from the main thread
- THEN the spinner frame is cleared, the message is written on its own line, and the
  spinner resumes — without corrupted terminal output

**Tests**: tests/test_in_out.py

---

### Requirement 4: Visibility Control

The spinner MUST support `hide()` and `show()` methods to temporarily suppress
animation while allowing custom terminal output, and a reentrant `hidden()` context
manager that nests correctly.

**Implementation**: yaspin/core.py:396 (`hide`), yaspin/core.py:443 (`show`),
yaspin/core.py:422 (`hidden`)

#### Scenario: Hide and show
- GIVEN a running spinner
- WHEN `spinner.hide()` is called
- THEN the spinner line is cleared and animation pauses
- WHEN `spinner.show()` is called
- THEN animation resumes

#### Scenario: Nested hidden contexts
- GIVEN a running spinner
- WHEN `hidden()` contexts are nested two levels deep
- THEN the spinner stays hidden until the outermost context exits

**Tests**: tests/test_in_out.py

---

### Requirement 5: Finalizers

The spinner MUST provide `ok(text)` and `fail(text)` methods that stop the spinner
and freeze a final static line (the last frame plus the provided text).

**Implementation**: yaspin/core.py:484 (`ok`), yaspin/core.py:489 (`fail`),
yaspin/core.py:515 (`_freeze`)

#### Scenario: Success finalizer
- GIVEN a running spinner
- WHEN `spinner.ok("✔ Done")` is called
- THEN the spinner stops and "✔ Done" is printed as a permanent line

#### Scenario: Failure finalizer
- GIVEN a running spinner
- WHEN `spinner.fail("✘ Error")` is called
- THEN the spinner stops and "✘ Error" is printed as a permanent line

**Tests**: tests/test_finalizers.py

---

### Requirement 6: Output Composition

The spinner MUST compose each output frame as `\r{frame} {text}{timer}` for
in-place updates, or `{frame} {text}{timer}\n` for the final frozen line. When
`side="right"` is set, the frame and text positions MUST be swapped.

**Implementation**: yaspin/core.py:593 (`_compose_out`)

#### Scenario: Left-side spinner (default)
- GIVEN `side="left"` (default)
- WHEN a frame is composed
- THEN the output is `\r<frame> <text>` (frame on the left)

#### Scenario: Right-side spinner
- GIVEN `side="right"`
- WHEN a frame is composed
- THEN the output is `\r<text> <frame>` (frame on the right)

**Tests**: tests/test_in_out.py, tests/test_properties.py

---

### Requirement 7: Text Truncation

When the combined width of frame, text, and timer exceeds the terminal width, the
text MUST be truncated and the configured ellipsis appended.

**Implementation**: yaspin/core.py:621 (`_compose_out` truncation logic),
yaspin/core.py:643 (`_get_max_text_length`)

#### Scenario: Long text with ellipsis
- GIVEN `ellipsis="..."` and a terminal width of 80
- WHEN the text would overflow the line
- THEN the displayed text is clipped and "..." is appended

**Tests**: tests/test_ellipsis.py

---

### Requirement 8: Timer Display

When `timer=True`, the spinner MUST display an elapsed-time counter formatted as
`(H:MM:SS.ff)` appended after the text.

**Implementation**: yaspin/core.py:614 (`_compose_out` timer block),
yaspin/core.py:331 (`elapsed_time` property)

#### Scenario: Timer enabled
- GIVEN `yaspin(timer=True)`
- WHEN the spinner is running
- THEN the output includes `(0:00:00.NN)` updated each frame

**Tests**: tests/test_timer.py

---

### Requirement 9: Fluent Interface

The `Yaspin` class MUST implement `__getattr__` to support fluent chaining of
spinner names, color names, highlight names, text attributes, and side settings
without explicit property assignments.

**Implementation**: yaspin/core.py:230 (`__getattr__`) — [ADR-0001](../../adr/0001-fluent-interface-via-getattr.md)

#### Scenario: Chained attributes
- GIVEN a `Yaspin` instance
- WHEN `spinner.cyan.bold.right` is accessed
- THEN `color="cyan"`, `attrs=["bold"]`, `side="right"` are all set and `self`
  is returned for further chaining

**Tests**: tests/test_attrs.py

---

### Requirement 10: Cursor Management

The spinner MUST hide the terminal cursor on start (via ANSI `\033[?25l`) and
restore it on stop (via `\033[?25h`) — but ONLY when writing to a TTY.

**Implementation**: yaspin/core.py:704 (`_hide_cursor`), yaspin/core.py:710 (`_show_cursor`)

#### Scenario: TTY cursor hide
- GIVEN a TTY stream
- WHEN the spinner starts
- THEN the cursor escape sequence is written and cursor disappears
- WHEN the spinner stops
- THEN the cursor is restored

#### Scenario: Non-TTY cursor skip
- GIVEN a non-TTY stream (e.g., StringIO)
- WHEN the spinner starts
- THEN no cursor escape sequences are written

**Tests**: tests/test_in_out.py
