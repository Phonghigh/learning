# SESSION 2026-09-15 — Understanding `@contextmanager` / `yield` / singleton in `shared_connection.py`

**Note:** This was a concept-learning tangent that came up while reviewing the Phase 2 refactor PR
(splitting simulation logic out of the PySide6 app into a Qt-free `app/engine_service/` package). No
code was changed in EWM-App during this session — it is a pure Q&A teaching session, logged because
the user explicitly asked for a durable record of the reasoning ("note lại kiến thức này thành 1 doc
cho tôi, tôi đã hiểu được rồi").

## 1. Requirement recap

While reviewing `app/database/shared_connection.py`, the user (in "Junior Developer Mode Level 1",
wants reasoning explained rather than answers alone) got stuck on this code and asked repeatedly,
across ~10 rounds, "tôi vẫn chưa hiểu... giải thích cho tôi" (I still don't understand... explain it
to me):

```python
def _ensure_engine() -> tuple[Engine, sessionmaker]:
    global _engine, _SessionLocal
    if _engine is None:
        _engine = create_engine(shared_database_url(), echo=False, pool_pre_ping=True)
        _SessionLocal = sessionmaker(
            bind=_engine, autoflush=False, autocommit=False, expire_on_commit=False
        )
    return _engine, _SessionLocal


@contextmanager
def get_shared_session():
    _, session_factory = _ensure_engine()
    session = session_factory()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

At the end the user said they now understood it and asked for it to be written up as a doc, then
redirected to log it in `E:\Learning` following that archive's own conventions instead of an ad-hoc
file.

## 2. How it was implemented + docs used

No implementation — this was pure explanation, built up incrementally because each answer revealed a
deeper unknown underneath it. The teaching order (see section 7) went: singleton pattern → `with`
statement surface behavior → why not use the raw `Session`'s own `with` support → what
`@contextmanager` actually transforms → `__enter__`/`__exit__` protocol from scratch → the "is
`close()` special?" question → the "why exactly twice" mechanical question about `next()` →
synthesis/recap tying back to the real codebase → a runnable executed demo (because description alone
wasn't landing) → final variable-name-vs-value confusion.

No external docs were fetched; the explanation used analogies (shared office printer, `open()`/file
handling) and a hand-written runnable demo script rather than official Python docs, since the user's
gaps were about the underlying execution model, not API usage.

## 3. Remember card (concept cheatsheet)

| Concept | What it is | Key line/example |
|---|---|---|
| Singleton pattern | Ensure only one instance of an object exists app-wide, backed by a module-level variable | `_engine: Engine \| None = None`; `if _engine is None: _engine = create_engine(...)` |
| `global` keyword | Without it, assigning to a module-level name inside a function creates a new local shadow instead of mutating the shared one — a silent bug | `global _engine, _SessionLocal` at the top of `_ensure_engine()` |
| `with X as Y:` | Guarantees `X.__enter__()` runs once before the block and `X.__exit__()` runs once after, even on exception | `with open("f.txt") as f: ...` auto-calls `f.__exit__` |
| `Session.__exit__` (raw SQLAlchemy) | Only calls `close()` — no auto-commit, no auto-rollback | Using `with SessionLocal() as s:` directly still needs manual `s.commit()` |
| `@contextmanager` | Decorator that turns a generator function (exactly one `yield`) into a class with `__enter__`/`__exit__`, without hand-writing that class | Code before `yield` = enter; the try/except/finally around `yield` = exit |
| `_GeneratorContextManager` (internal) | The class `@contextmanager` actually builds; its `__enter__` calls `next(gen)` once, its `__exit__` calls `next(gen)` (or `gen.throw()`) once more | Exactly 2 `next()` calls total — a 2nd `yield` triggers `RuntimeError: generator didn't stop` |
| `yield` vs `return` | `yield` pauses the function and hands a *value* back to the caller, keeping local state alive to resume later; `return` ends the function | `yield session` hands the live `Session` object to the `with ... as session:` caller |
| Variable name vs. value | The name on each side of `yield`/`return` is arbitrary — only the object it points to matters | Renaming both sides (`cai_object_ket_noi_db`, `bien_ngoai_kia`) behaves identically |

## 4. Code changes in detail

No production code changed. The concrete artifact of this session is a runnable demo script written
to prove the enter/body/exit ordering, since verbal description alone was not enough for the user
(round 9). It lived at a job-scratch temp path, **not part of the EWM-App repo**:
`C:\Users\Admin\.claude\jobs\d5422779\tmp\demo_yield_session.py` — included here for the record only.

### 1. Runnable proof of `get_shared_session()` behavior — demo script (temp, not committed)

**Demo script (paraphrased structure, matches what was actually run):**
```python
class FakeSession:
    def execute(self, *a):
        print("  [session] execute() called")
    def commit(self):
        print("  [session] commit() called")
    def rollback(self):
        print("  [session] rollback() called")
    def close(self):
        print("  [session] close() called")

from contextlib import contextmanager

@contextmanager
def get_shared_session():
    print("BEFORE yield: creating session")
    session = FakeSession()
    try:
        yield session
        print("AFTER yield, success path: about to commit")
        session.commit()
    except Exception:
        print("AFTER yield, exception path: about to rollback")
        session.rollback()
        raise
    finally:
        print("FINALLY: about to close")
        session.close()

print("=== CASE 1: success ===")
with get_shared_session() as session:
    print("  INSIDE with-block: doing work")
    session.execute("SELECT 1")

print("\n=== CASE 2: exception ===")
try:
    with get_shared_session() as session:
        print("  INSIDE with-block: about to raise")
        raise ValueError("boom")
except ValueError:
    print("caught ValueError outside the with-block")
```

**Real captured output (first attempt — failed):**
```
UnicodeEncodeError: 'charmap' codec can't encode character '\u1ed7' in position ...
```

**Real captured output (second attempt, `PYTHONUTF8=1 python demo_yield_session.py` — succeeded):**
```
=== CASE 1: success ===
BEFORE yield: creating session
  INSIDE with-block: doing work
  [session] execute() called
AFTER yield, success path: about to commit
  [session] commit() called
FINALLY: about to close
  [session] close() called

=== CASE 2: exception ===
BEFORE yield: creating session
  INSIDE with-block: about to raise
AFTER yield, exception path: about to rollback
  [session] rollback() called
FINALLY: about to close
  [session] close() called
caught ValueError outside the with-block
```

**What changed:** N/A (no diff — this is new scratch code, not a modification of any repo file).
**Why:** The user could not build a mental model of "before yield vs after yield vs exception path"
from description alone; running the demo and reading real printed output line-by-line, matched back
to which part of `get_shared_session()` produced each line, is what closed the gap (their own words:
"tôi đã hiểu được rồi" came right after this).
**How it behaves now:** The user can now point at any line of `get_shared_session()` in
`shared_connection.py` and say which of the two demo output blocks (success path vs exception path)
it corresponds to.

## 5. How to find this again

- File under review: `app/database/shared_connection.py` (functions `_ensure_engine`,
  `get_shared_session`)
- grep terms: `@contextmanager`, `get_shared_session`, `_ensure_engine`, `sessionmaker`
- Python stdlib concept: `contextlib.contextmanager`, `contextlib._GeneratorContextManager`
- Related PR: Phase 2 refactor — Qt-free `app/engine_service/` package extraction (this Q&A was a
  tangent during that review, not part of its diff)

## 6. Concepts introduced

- **Singleton pattern**: guarantee exactly one shared instance of a resource (here: one DB `Engine`
  = one connection pool) across the whole app's lifetime, done via a module-level variable plus a
  `None`-check. Needed here because opening a new `Engine`/connection pool on every call would leak
  connections and defeat pooling.
- **`global` keyword**: without it, assignment inside a function creates a local variable shadowing
  the module-level one, silently breaking the singleton (no error raised — just wrong behavior).
  Needed to explain why `_ensure_engine()` declares `global _engine, _SessionLocal` at the top.
- **`with` statement `__enter__`/`__exit__` contract**: `with X as Y:` is a fixed protocol — Python
  calls `X.__enter__()` once, binds its return value to `Y`, runs the block, then calls
  `X.__exit__()` exactly once whether or not an exception occurred. Needed because the whole point of
  `get_shared_session()` is to hook custom cleanup logic into that guaranteed second call.
- **`@contextmanager`**: converts a generator function with exactly one `yield` into an object
  satisfying the `__enter__`/`__exit__` contract, so the code doesn't need a hand-written class.
  Needed to explain why `get_shared_session()` is written as a generator instead of a class.
- **Generator functions / `yield` vs `return`**: `yield` pauses execution and hands back a value
  while keeping the function's state alive to resume later on the next `next()` call; `return` ends
  the function permanently. Needed because `get_shared_session()`'s pause-at-`yield` is exactly what
  lets the code before `yield` act as setup and the code after it act as teardown.
- **Variable name vs. the value it references**: names in Python are just labels pointing at objects;
  `yield <name>` passes the object, not the name — the caller's `as <name>` on the other side can use
  a totally different name for the same object. Needed because the user assumed the identical name
  `session` on both sides of `yield` was structurally required.

## 7. Where it got stuck (in order)

1. **Symptom:** User did not recognize the singleton pattern in `if _engine is None: ...`.
   **Cause:** No prior exposure to the pattern or to module-level state.
   **Fix:** Explained via a "one shared office printer" analogy — everyone in the office reuses the
   same printer instead of buying a new one each time; `_engine is None` is the "has anyone already
   bought a printer?" check. Flagged the `global` beginner-trap explicitly: forgetting it makes the
   assignment create an invisible local variable, so the function looks correct but silently never
   updates the shared `_engine`.

2. **Symptom:** User's first-pass understanding of `@contextmanager`/`with` was surface-level only
   ("it auto-closes files, like `open()`").
   **Cause:** Only knew the by-example behavior, not the underlying mechanism.
   **Fix:** Contrasted against manual `try/finally`, and mapped code-before-`yield` → setup,
   code-after-`yield` (in try/except/finally) → teardown, using `open()` as the familiar anchor.

3. **Symptom:** User asked why not just use `with` directly on the raw SQLAlchemy `Session` object,
   since it already supports `with` itself.
   **Cause (this was the real misconception):** Assumed `Session.__exit__` behaves like a "smart"
   context manager that commits or rolls back automatically.
   **Fix:** Pointed out concretely that `Session.__exit__` only calls `close()` — no commit, no
   rollback. Demonstrated that using the raw `Session` directly would force every call site to
   hand-write `try/commit/except/rollback/finally/close`, and forgetting `session.commit()` would be
   a silent bug (code runs with no error, but nothing is persisted to the DB). `get_shared_session()`
   exists specifically to centralize that policy once (DRY) instead of repeating it everywhere.

4. **Symptom:** User asked "does `@contextmanager` exist just so we can write `yield session`?" —
   assuming the decorator's only job is to allow that one line.
   **Cause:** Framing collapsed the whole-function transformation down to a single line.
   **Fix:** Corrected: `@contextmanager` transforms the entire function — everything before `yield`
   becomes `__enter__`, and the try/except/finally wrapped *around* `yield` becomes `__exit__` — not
   just the `yield` line in isolation.

5. **Symptom:** User admitted they didn't actually understand what `with X as Y:` does mechanically
   at all, underneath the by-example explanations so far.
   **Cause:** Prior explanations had stayed at the analogy level; user needed the literal mechanism.
   **Fix:** Went back to fundamentals — showed the informal desugaring of `with X as Y: BODY` into
   direct `__enter__()`/`__exit__()` calls, then wrote a from-scratch `class MyFile` with explicit
   `__enter__`/`__exit__` methods to prove there is no hidden magic: `with` just guarantees calling a
   fixed pair of method names, and the class author decides what code goes inside them.

6. **Symptom:** User asked how `with` "knows" to call `f.close()` — assumed this specific behavior was
   special-cased by Python.
   **Cause:** Confusing "Python always calls `__exit__`" (true, generic) with "Python knows to call
   `close()` specifically" (false — that's just what `open()`'s author wrote inside their own
   `__exit__`).
   **Fix:** Reinforced that `__exit__` is an ordinary method name Python's `with` is contractually
   obligated to call once; whoever wrote `open()` chose to put `self.close()` inside that method —
   nothing about `close()` itself is privileged.

7. **Symptom:** User asked why `with` (via `@contextmanager`) calls `next()` on the generator exactly
   twice, and why exactly one `yield` is required.
   **Cause:** Hadn't yet seen how `_GeneratorContextManager` (the class `@contextmanager` builds)
   implements `__enter__`/`__exit__` in terms of the generator's `next()`.
   **Fix:** Separated two independent facts: (a) `with` itself always calls `__enter__` once and
   `__exit__` once — fixed by the `with` statement's own contract, unrelated to generators; (b)
   `contextlib`'s generated class happens to implement `__enter__` as one `next(gen)` call and
   `__exit__` as one more `next(gen)` (or `gen.throw(...)` on exception). Showed simplified pseudocode
   of `_GeneratorContextManager`. Explained that a second `yield` in the generator raises
   `RuntimeError: generator didn't stop`, because `contextmanager` never calls `next()` a third time
   to drain it.

8. **Symptom:** User asked a recap/synthesis question — "why `yield session` specifically, why need
   `@contextmanager` here, what is this function's purpose overall" — signaling the pieces hadn't yet
   connected into one coherent picture.
   **Cause:** Each prior answer addressed one mechanism in isolation; nothing had tied it back to why
   *this specific codebase* needs it.
   **Fix:** Answered by grounding in the real project: `session` is yielded because it's the actual
   object callers need (to call `.execute()` etc.); `@contextmanager` is the shortest way to get
   `__enter__`/`__exit__` behavior without a hand-written class; and the concrete purpose is
   centralizing the "commit-on-success / rollback-on-exception / always-close" unit-of-work policy in
   one place, since Phase 3+ of this project (`.inp` import, crawler, simulation result writes) will
   have many DB call sites that would otherwise each need to repeat this logic.

9. **Symptom:** User said they still didn't understand `yield session` and asked for an example of
   the function's actual runtime behavior, not another description.
   **Cause:** Description-only explanations had exhausted their usefulness for this user; needed
   empirical proof.
   **Fix:** Wrote and **actually ran** a demo script with a `FakeSession` class (print-instrumented
   `execute`/`commit`/`rollback`/`close`) wrapped in a copy of the real `get_shared_session()` logic,
   covering both the success path and the exception path (see section 4 for the verbatim script and
   output).
   **Real runtime error hit along the way (not inferred — directly observed):** first execution
   failed with `UnicodeEncodeError: 'charmap' codec can't encode character '\u1ed7' in position ...`.
   Root cause: Windows console defaults to the `cp1252` codepage for stdout, not UTF-8, and the demo
   printed Vietnamese text containing diacritics (`\u1ed7` is `ỗ`). Fixed by re-running with the
   `PYTHONUTF8=1` environment variable prefix, which forces UTF-8 I/O mode regardless of the console
   codepage. Second run succeeded and produced the output captured verbatim in section 4.

10. **Symptom:** User was confused that the generator's internal variable name (`session`) and the
    caller's `with ... as session:` variable name were identical, and thought this name-matching was
    load-bearing or somehow "magic" — as if `yield` worked by matching names.
    **Cause:** Had not yet separated "the name a variable is written with" from "the object it points
    to" as two independent things.
    **Fix:** Corrected: these are two independent variables in two different scopes that merely
    happen to share a name by convention/readability; what actually flows through `yield` is the
    *value* (the real `Session` instance from `session_factory()`), never the identifier. Proved it
    by renaming one side to `cai_object_ket_noi_db` (inside the generator) and the other to
    `bien_ngoai_kia` (at the call site) and confirming identical behavior. Reinforced with a simpler,
    non-DB example: `def tao_list(): danh_sach = [1,2,3]; return danh_sach` then `x = tao_list()` —
    `return`/`yield` pass the list object, not the name `danh_sach`.

## 8. Verify

The demo script in section 4 was **actually executed twice**, not just described:

1. First run: `python demo_yield_session.py` → failed with
   `UnicodeEncodeError: 'charmap' codec can't encode character '\u1ed7' in position ...` (real,
   observed error from Windows' cp1252 console encoding meeting Vietnamese diacritics in the print
   statements).
2. Second run: `PYTHONUTF8=1 python demo_yield_session.py` → succeeded, producing the verbatim
   two-case output (`=== CASE 1: success ===` / `=== CASE 2: exception ===` blocks) quoted in full in
   section 4. Reading that output line-by-line against `get_shared_session()`'s source (before-yield
   / body-of-with / after-yield success path / after-yield exception path / finally) is what confirmed
   the user's understanding was now correct.

No production tests apply — no repo code changed.

## 9. Gotchas

- If `shared_connection.py` is ever touched again to remove the singleton (`_engine`), remember any
  new implementation must still guard against the same silent-shadowing bug: forgetting `global` (or
  its equivalent) makes assignment create a local variable instead of mutating shared state, with no
  error raised.
- Anyone extending `get_shared_session()` must keep exactly **one** `yield` — adding a second one (for
  example, to "yield twice" for some retry logic) will raise `RuntimeError: generator didn't stop`,
  because `@contextmanager`'s generated `__exit__` only ever calls `next()` once after the first
  `yield`.
- On Windows, any ad-hoc debug/demo script that prints non-ASCII (Vietnamese, accented, etc.) text to
  the console should run with `PYTHONUTF8=1` (or otherwise force UTF-8 stdout) to avoid the same
  `UnicodeEncodeError` hit in round 9 — this is a console codepage issue, not a bug in the script
  logic itself.
- The demo script used for this session was scratch-only, living under a job-temp path
  (`C:\Users\Admin\.claude\jobs\d5422779\tmp\demo_yield_session.py`); it was never part of the EWM-App
  repo and should not be assumed to exist if this log is read later.
