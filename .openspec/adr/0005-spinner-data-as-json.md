# ADR-0005: Spinner Definitions as Bundled JSON Data

**Status**: Accepted

**Date**: 2026-02-22 (reverse-engineered from codebase)

**Decision ID**: 0005-spinner-data-as-json

---

## Context

Yaspin ships 80+ named spinner animations. Each spinner is a `{frames, interval}`
pair. These could be defined as Python constants, a Python dict, or an external data
file. The community source for these spinners is the popular `cli-spinners` npm
package (which uses JSON).

## Decision

Spinner definitions are stored in `yaspin/data/spinners.json`, loaded at module
import time via `pkgutil.get_data(__name__, "data/spinners.json")`. The JSON is
parsed with a `object_hook` (`_hook`) that converts each object into a `namedtuple`,
resulting in a single `Spinners` object whose attributes are the spinner namedtuples.

The list of valid spinner names (`SPINNER_ATTRS`) is maintained separately in
`yaspin/constants.py` and used by `Yaspin.__getattr__` for attribute dispatch.

## Consequences

### Positive
- Easy to update the spinner library: only the JSON file and `SPINNER_ATTRS` list
  need changes — no Python code changes required
- JSON format matches the upstream `cli-spinners` source, making it easy to sync
- `namedtuple` provides attribute-style access (`Spinners.dots.frames`) for free
- `pkgutil.get_data` correctly loads the file whether the package is installed as a
  directory or a zip (wheel)

### Negative
- `SPINNER_ATTRS` list in `constants.py` must be kept in sync with JSON keys manually
- `namedtuple` attribute names come from JSON keys; if a key changes, existing code
  using the old attribute name breaks silently (no type-checking)
- Loading at import time adds a small startup cost (JSON parse of ~12KB)

### Neutral
- The `Spinners` object is accessed lazily from `Yaspin.__getattr__` (imported only
  when a spinner name is first resolved), limiting startup impact

## Related Specs

- [006-spinner-collection](../specs/006-spinner-collection/spec.md) — Req 1: Built-in Spinner Library, Req 2: Spinners Object Construction
