# PLAN.md

## Solution plan

**Issue:** review_service unit tests misconfigure async mocks — 13 of 19 tests fail ([#158](https://github.com/ascherj/pathreview/issues/158))

### Understand

**Root cause:** the tests build their fake DB session by setting `mock_result = AsyncMock()` for the object `db.execute(...)` returns. `AsyncMock`'s auto-created child attributes are themselves `AsyncMock`, so calling `mock_result.scalars()` returns an unawaited coroutine instead of a real result-like object, and chaining `.first()` or `.all()` onto that coroutine raises `AttributeError`.

**Expected vs. actual:**
- *Expected:* `db.execute()` is the only real async boundary in SQLAlchemy's async API. Once awaited, its return value (`Result`) has plain synchronous `.scalars()`, `.first()`, and `.all()` methods — so a correct mock only needs `AsyncMock` on `execute()` itself, with a regular `Mock`/`MagicMock` standing in for the `Result` it returns.
- *Actual:* the returned mock object was itself `AsyncMock`, so `.scalars()` returned a coroutine, and `.first()`/`.all()` blew up before any test assertion ever ran. 13 of 19 tests failed this way; the service code in `core/services/review_service.py` was never the problem.

### Map

Files/functions involved:
- **`tests/unit/test_review_service.py`** — every failing test's `mock_db_session`/`mock_result` setup. This is the only file that needs a functional change.
- **`core/services/review_service.py`** — `get_review()`, `list_reviews()`, `create_review()`. Read-only reference: used to confirm exactly which chain (`.first()` vs `.all()`) and how many `execute()` calls each function actually makes, so the mocks can be checked against real behavior rather than guessed at.
- No other module is involved — the bug is entirely test-construction, not application logic.

### Plan

1. Reproduce first: run `pytest tests/unit/test_review_service.py -q` before touching anything, and confirm the failure signature matches the issue exactly (13 failed, 6 passed, `AttributeError: 'coroutine' object has no attribute 'first'`/`'all'`).
2. Read `review_service.py` to establish ground truth for the async/sync boundary — confirm `db.execute()` is awaited and everything chained after it (`.scalars()`, `.first()`, `.all()`) is synchronous, so I know what the mocks *should* look like before changing anything.
3. In each of the 13 failing tests, change `mock_result = AsyncMock()` to `mock_result = Mock()`, leaving `mock_db_session.execute = AsyncMock(return_value=mock_result)` untouched — `execute()` stays properly async, its return value stops being falsely async.
4. Re-run the suite and handle any remaining failures individually rather than assuming the blanket swap fixes everything — e.g. one test asserted `execute.assert_called_once()` even though `list_reviews()` correctly calls `execute()` twice by design (a count query, then the paginated query); that's a wrong assertion, not a mock problem, and needs its own fix.
5. Run the full test suite (not just this file) plus lint/type checks to confirm no regressions elsewhere and that the fix doesn't mask a different bug.

### Inputs & outputs

- **Input:** the existing test file with its current (broken) mock setup. No production code inputs change.
- **Output:** the same 19 tests, passing, actually exercising `get_review`, `list_reviews`, and `create_review` — verifying real behavior (correct-owner matching, pagination, dedup via `AlreadyInWatchlistError`-style checks, ordering) instead of erroring out on a mock artifact before any assertion runs.

### Risks & unknowns

- Blindly swapping `AsyncMock()` → `Mock()` everywhere could paper over a genuinely different bug in a specific test if that test's assertions were only "passing" by accident of the crash — each test needs its assertions re-checked for meaning, not just for absence of an exception.
- Not all 13 failures are guaranteed to share the identical root cause going in — needs verifying failure-by-failure rather than assuming one fix covers all of them (it turned out all 13 did share the same cause, but that was confirmed, not assumed).
- Unclear up front whether any fix would be needed in `review_service.py` itself — resolved by confirming the service's async usage was already correct per SQLAlchemy's async API before changing any tests.

### Edge cases

- `get_review()` returning `None` (wrong user / nonexistent review) vs. returning a real review object — both branches of `.first()` (`None` vs. a `Mock` instance) need to work under the corrected mock.
- `list_reviews()` pagination: an empty result set (`.all()` → `[]`) vs. a populated one, and the fact that `list_reviews()` issues **two** separate `execute()` calls (count query + paginated query) — any test asserting call counts must reflect that, not assume a single call.
- Tests must mock the *specific* chain the function under test actually uses — `get_review()` calls `.scalars().first()`, while `list_reviews()` calls `.scalars().all()` (twice) — a mock configured for the wrong chain would silently return the wrong shape of data even without raising an exception.
