I added the usage-log guard and the opt-ins. Two checks are still open: the base-tree check and the real `logs/usage/` before/after count. The sandbox blocked the compound shell command for each, so I did not run them.

**What I ran**
- `.venv/bin/python -m pytest tests/unit/test_usage_log_guard.py -q` gave 6 passed.
- `.venv/bin/python -m pytest tests/ -q` gave 874 passed, 17 skipped, 8 deselected, 1 xfailed. The strict xfail is still xfailed, and nothing failed or errored. No test outside the plan's list needed an opt-in, so I left the four contingent modules untouched.

**Not run**
- **Base-tree check:** I did not confirm that the guard tests fail on `6d3dd93` with `DID NOT RAISE`. The `guard-test-fails-on-base-tree` command still needs running.
- **Line-count check:** I did not count the lines in `logs/usage/` before and after the suite. Run `usage-log-unchanged-by-suite` to confirm.
- **Live tier and the other criteria:** I did not run `tests/live --collect-only -m live`, the release-note shape check, the product-code diff check or the `test_usage.py` recording run.

**What changed**
- `tests/conftest.py`: a session-wide guard around `harness.usage.append_jsonl`.
  - **Before any write:** it resolves the target directory and calls `pytest.fail` if that is the real `logs/usage/` or a folder inside it. The message names both directories, cites ADR-021 and names the fix.
  - **Protected paths:** captured when conftest is imported, so a monkeypatch can't move them.
  - **Fixture:** it also adds the non-autouse `usage_log_dir` fixture, which sends the recorder to `tmp_path / "usage"` and returns that path.
- `tests/unit/test_usage_log_guard.py` (new): tests for `record_run` and a direct `append_jsonl` call with no `log_dir`, and for `REAL/x/..` and a subdirectory. They also cover a monkeypatch back to the real directory. Each asserts a read-only snapshot of the real directory is unchanged. Two positive controls show an explicit `tmp_path` and the `usage_log_dir` fixture still write normally.
- `tests/unit/test_agent_loop.py`, `test_run_cli.py`, `test_live_tier.py`, `tests/live/test_live_eval.py`: each gets a module-level `usage_log_dir` opt-in marker. In the live file it sits alongside the existing `live` mark.
- `releases/_next/tests-never-write-usage-log.md` (new): a patch release note with the before/after count in its Verification block. The PR number is a `#TBD` placeholder.

No product files changed, and the existing files in `logs/usage/` were left alone.
