# python-fundamentals

Core Python language mechanics — generators, `yield`, context managers, decorators, scoping, and
other execution-model concepts that come up while reading or reviewing real project code.

| Date | Project | Title | What broke / what changed |
|---|---|---|---|
| 2026-09-15 | EWM-App | [Understanding `@contextmanager` / `yield` / singleton in `shared_connection.py`](SESSION-2026-09-15-EWM-App-context-manager-yield-teaching.md) | Not a bug — a 10-round conceptual Q&A during a Phase 2 PR review where the user (Junior Developer Mode) got stuck on `get_shared_session()`'s `@contextmanager`/`yield` pattern and the `_ensure_engine()` singleton. Built up understanding layer by layer: singleton pattern via `global` + module-level `None` check, why `with` on a raw SQLAlchemy `Session` isn't enough (`Session.__exit__` only closes, never commits/rolls back), what `@contextmanager` actually transforms (whole function, not just the `yield` line), the `__enter__`/`__exit__` protocol from scratch, why `next()` is called exactly twice on the generator, and a final mix-up between a variable's name and the value it references. Closed the gap with an actually-executed demo script (`FakeSession` class) proving the enter/body/exit ordering for both success and exception paths — hit a real `UnicodeEncodeError` from Windows cp1252 console encoding on the first run, fixed with `PYTHONUTF8=1`. No production code changed. |
