# CLAUDE.md

Desktop app for cataloguing personal belongings in nested categories, with editable tables and a calendar of worn/bought/stored/discarded events. **The full spec is in `SPEC.md` — read it before starting any task.**

## Stack
- Python 3.11+, tkinter / ttk (no other GUI toolkit)
- SQLite via stdlib `sqlite3` (no ORM)
- Pillow for images
- pytest for tests

## Commands
- Run: `python -m categories`
- Test: `pytest`
- Install: `pip install -e .`

## Workflow: one phase at a time
Build strictly in the phase order from `SPEC.md` §9. Every phase follows these steps:

1. **Plan.** List the `db.py` functions the phase needs, then the service and UI work.
2. **Stub the database layer.**
   - Add each new function to `db.py` as a stub: a type-hinted signature, a docstring describing arguments, return value and edge cases, and `raise NotImplementedError` as the body.
   - Write **no SQL** and give **no hints** about how to write the query.
   - Add pytest tests for each stub in `tests/test_db.py`.
   - Skip this step if the phase needs no new queries.
3. **Stop.** List the stubs to implement and wait. I write the SQL myself.
4. **When I say continue:** build the services and UI on top of the stubs, run `pytest`, and tell me what to click to verify the phase by hand.
5. **Commit and push.**
   - Once the tests pass, run `git add -A`, commit with the message `Phase N: <name>`, then `git push`.
   - If no remote is set or the push fails, report it. Don't create a remote and don't force-push.
6. **Stop.** Wait for an explicit "continue" before starting the next phase. Don't start, scaffold or "prepare" the next phase.

Other rules:
- **Never write or edit SQL in `db.py`**, including the schema from `SPEC.md` §7. If one of my queries fails a test, point to the failing test and explain the problem in words. Only fix it if I ask.
- The app must be runnable at the end of every phase.
- If something in the spec is ambiguous or seems wrong, ask instead of guessing. Check `SPEC.md` §11 (Open questions) before deciding anything listed there.
- Keep changes scoped to the current phase. Don't refactor unrelated code.
- I know tkinter and SQL, so skip explanations of the basics.

## Code conventions
- **4-space indentation, always.**
- PEP 8, type hints on all function signatures, dataclasses for models.
- **All SQL lives in `db.py`.** UI modules never touch the connection directly.
- Business rules (e.g. "putting an item into storage logs a `stored` event and sets status") live in `services.py`, not in UI callbacks.
- Always use parameterised queries (`?`), never string formatting for SQL.
- Dates are stored as ISO `YYYY-MM-DD` strings.
- Refer to built-in event types by their `key` (`bought`, `discarded`, `stored`, `worn`), never by id or name. Names are user-editable.
- One event can involve many items (`event_item` join table). Never create one event per item for a manual log.
- UI components are classes subclassing `ttk.Frame` (or `tk.Toplevel` for popups). They communicate upward via callbacks or virtual events (`<<CategorySelected>>`, `<<DataChanged>>`), not by reaching into each other.
- Any change to items or events fires `<<DataChanged>>` on the root window; the title count, table and calendar refresh on it.

## tkinter gotchas
- `PhotoImage` objects are garbage-collected if no Python reference is kept — store them in a cache dict (see `images.py`), or thumbnails will silently disappear.
- `ttk.Treeview` supports only one image per row (in column `#0`), and has no built-in cell editing. Inline editing is done by placing an `Entry` over the cell using `tree.bbox(item, column)` (see `editable_tree.py`).
- tkinter is not thread-safe — don't touch widgets from other threads.
- Run `PRAGMA foreign_keys = ON` on every new SQLite connection; it's off by default.
- Use a `ttk.Style` for theming; don't hard-code colours in individual widgets.

## Testing
- Test `db.py` and `services.py` with pytest against an in-memory SQLite DB (`:memory:`).
- Must-cover cases:
  - descendant item queries
  - column inheritance/union
  - search matching (name, field values, location; both scopes)
  - multi-item events
  - applicable event types for a mixed selection
  - automatic `bought`/`discarded` events and their status side effects
  - deleting an item that's the last one in an event
  - title count excluding discarded items
- UI is checked manually — describe what to click to verify each phase.
