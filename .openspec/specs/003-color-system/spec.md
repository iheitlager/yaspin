---
domain: color-system
version: 3.4.0
status: draft
date: 2026-02-22
---

# Color System

## Overview

The Color System manages ANSI terminal color application to spinner frames. It
supports foreground colors, background highlights, and text attributes (bold, blink,
etc.), composed into a single callable via `functools.partial`. Colors are
automatically disabled for non-TTY streams to avoid literal ANSI escape codes in
redirected output.

## Philosophy

- Colors MUST be applied only to the **frame** character(s), not the text.
- Color composition is deferred to a `_color_func` partial, re-built whenever any
  color-related property changes.
- Non-TTY environments silently disable colors to keep output clean.
- The fluent `__getattr__` interface allows chaining like `spinner.cyan.bold`.

## Requirements

### Requirement 1: Foreground Color

The spinner MUST support setting a foreground color via the `color` property or
the `color` constructor argument. Valid values are those accepted by `termcolor.colored`.

**Implementation**: yaspin/core.py:277 (`color` property), yaspin/core.py:724 (`_set_color`)

#### Scenario: Valid color
- GIVEN `yaspin(color="cyan")`
- WHEN a frame is composed
- THEN the frame is wrapped in the cyan ANSI escape sequence

#### Scenario: Invalid color
- GIVEN `yaspin(color="neon-purple")`
- WHEN the spinner is initialized
- THEN a `ValueError` is raised listing valid color options

**Tests**: tests/test_attrs.py, tests/test_in_out.py

---

### Requirement 2: Background Highlight

The spinner MUST support setting a background highlight via the `on_color` property
or constructor argument.

**Implementation**: yaspin/core.py:285 (`on_color` property), yaspin/core.py:734 (`_set_on_color`)

#### Scenario: Valid highlight
- GIVEN `yaspin(on_color="on_blue")`
- WHEN a frame is composed
- THEN the frame has the blue background ANSI sequence applied

#### Scenario: Invalid highlight
- GIVEN `yaspin(on_color="on_neon")`
- WHEN the spinner is initialized
- THEN a `ValueError` is raised

**Tests**: tests/test_attrs.py

---

### Requirement 3: Text Attributes

The spinner MUST support one or more text attributes (bold, dark, underline, blink,
reverse, concealed) via the `attrs` property or constructor argument. Multiple attrs
MUST be combined additively.

**Implementation**: yaspin/core.py:294 (`attrs` property), yaspin/core.py:300 (union merge)

#### Scenario: Single attribute
- GIVEN `yaspin(attrs=["bold"])`
- WHEN a frame is composed
- THEN the bold ANSI sequence is applied to the frame

#### Scenario: Additive chaining
- GIVEN `spinner.bold.underline`
- WHEN attrs are accessed
- THEN both "bold" and "underline" are in `spinner.attrs`

**Tests**: tests/test_attrs.py

---

### Requirement 4: Color Disabled for Non-TTY

When the output stream is not a TTY, the spinner MUST NOT apply ANSI color codes.
The `_color_func` MUST be `None` in that case.

**Implementation**: yaspin/core.py:574 (`_compose_color_func`), yaspin/core.py:494 (`_supports_ansi_codes`)

#### Scenario: StringIO stream (non-TTY)
- GIVEN `yaspin(color="green", stream=io.StringIO())`
- WHEN a frame is composed
- THEN the frame is the raw character without escape codes

**Tests**: tests/test_in_out.py, tests/test_stream.py

---

### Requirement 5: Color Function Recomposition

Whenever `color`, `on_color`, or `attrs` is changed (via property setters or
`__getattr__` chaining), the `_color_func` MUST be immediately recomposed.

**Implementation**: yaspin/core.py:282 (color setter), yaspin/core.py:290 (on_color setter),
yaspin/core.py:299 (attrs setter)

#### Scenario: Dynamic color change
- GIVEN a running spinner with `color="red"`
- WHEN `spinner.color = "blue"` is set
- THEN subsequent frames are rendered in blue

**Tests**: tests/test_attrs.py
