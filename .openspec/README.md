# OpenSpec — yaspin

This directory contains the behavioral specifications and architectural decision records
for **yaspin**, a lightweight terminal spinner library for Python.

## Structure

```
.openspec/
├── specs/          # Behavioral specifications (current implemented state)
│   ├── 001-spinner-engine/    # Core Yaspin class, threading, lifecycle
│   ├── 002-public-api/        # Factory functions and decorator support
│   ├── 003-color-system/      # ANSI color composition and attribute chaining
│   ├── 004-stream-handling/   # SafeStreamWrapper, TTY detection
│   ├── 005-signal-handling/   # POSIX signal registration and cleanup
│   └── 006-spinner-collection/ # Built-in spinner data and custom spinners
├── adr/            # Architectural Decision Records
└── changes/        # Delta specs (proposed changes)
```

## Conventions

- Requirements use RFC 2119 keywords: **MUST**, **SHOULD**, **MAY**
- Each requirement includes `**Implementation**: <path>` pointing to the source
- Scenarios use GIVEN-WHEN-THEN format derived from tests and usage examples
- ADRs are referenced from requirements as `ADR-XXXX`

## Traceability

Specs were bootstrapped from codebase analysis using code-analyzer (issues #266, #267).
Traceability edges:
- **IMPLEMENTED_BY**: `**Implementation**: src/path` links in requirements
- **ADDRESSED_BY**: `ADR-XXXX` references in Implementation lines
- **TESTED_BY**: declared via `**Tests**: tests/path` on scenarios

## Status

Bootstrapped on 2026-02-22 from yaspin v3.4.0 (commit `8062faf`).
All specs are **draft** pending review.
