I moved the usage-log guard so it installs when `tests/conftest.py` is imported, which is before any test module is collected. The two usage test files pass (16 tests). I did not run the full suite, the live-tier collect check, or the base-tree `DID NOT RAISE` check.

- **`tests/conftest.py`:**
  - The guard now replaces `harness.usage.append_jsonl` at conftest import. That covers the module attribute, `from harness.usage import append_jsonl` in a test module, and `record_run`'s internal call.
  - A `pytest_unconfigure` hook puts the original back when the session ends.
  - The guard's behaviour and message are unchanged, as are the protected directories captured at import and the `usage_log_dir` fixture.
  - The old session-scoped autouse fixture is gone. It ran after collection, which is why a from-import could capture the unguarded function.
- **`tests/unit/test_usage_log_guard.py`:** I added `test_a_from_imported_append_jsonl_is_also_stopped`. It uses a from-import made at module import time and calls it with no `log_dir`. It asserts the failure text names the real directory and that the snapshot of `logs/usage/` is unchanged. On the base tree it should fail with `DID NOT RAISE`, since nothing there raises.
