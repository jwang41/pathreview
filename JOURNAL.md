# Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/158

**Issue title:** review_service unit tests misconfigure async mocks — 13 of 19 tests fail

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
`tests/unit/test_review_service.py` builds its fake database session by setting `mock_result = AsyncMock()` for the object that `db.execute(...)` returns. Because an `AsyncMock`'s auto-created child attributes are themselves `AsyncMock`, calling `mock_result.scalars()` returns an unawaited coroutine instead of a real result object, so chaining `.first()` or `.all()` onto it fails with `AttributeError: 'coroutine' object has no attribute 'first'` (or `'all'`). The actual service code in `core/services/review_service.py` is correct — `db.execute()` is the only real async boundary; its return value is a plain synchronous `Result` with synchronous `.scalars()`/`.first()`/`.all()` methods — so this is purely a test-construction bug, not a service bug. A successful fix makes the mock session use `AsyncMock` only for `execute()` itself and a plain `Mock`/`MagicMock` for the result it returns, so all the existing CRUD tests for `get_review`, `list_reviews`, and `create_review` actually exercise the code instead of erroring out before any assertion runs.

**Branch name:** fix/158-review-service-unit-tests-misconfigure-async-mocks

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger
