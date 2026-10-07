# Implementation Plan

## Bounded objective

Make the default test run (`.venv/bin/python -m pytest tests/ -q`) unable to write to the real usage log directory, `logs/usage/`.

- A guard installed by `tests/conftest.py` stops any test that sends the usage recorder to the real directory. The test fails, and the failure names the directory it tried to write to.
- Tests that legitimately record usage keep doing so, against a temporary directory.
- A new test shows the guard working. That test fails on base commit `6d3dd93`.

No product code changes: `harness/**`, `config/**`, `evals/**` and the record format stay untouched. Deleting or cleaning the files already in `logs/usage/` is not part of this work.

## Definition Pack interpretation

- **ADR-021 (`docs/decisions.md`, authority 1)**, last paragraph: "Tests never write to `logs/usage/`." This is the rule being enforced. The rest of ADR-021 (the pipeline writing request and summary records, the `schema_version` bump) is a separate obligation and not part of this item.
- **ADR-005** says telemetry is written to `logs/usage/usage-YYYY-MM-DD.jsonl`, and `record_run()` never raises. That second point matters here: a guard that raises an ordinary `Exception` inside `append_jsonl` would be caught by `record_run`'s `except Exception` and logged. The test would still pass. The guard therefore has to raise something that is not an `Exception`.
- **`05_measurements.md` and `01_orientation.md` C2 (authority 9, evidence):** "`logs/usage/` is contaminated by the unit test suite: 2026-09-29 has 468 lines, all `function:*` test models." `function:*` is the model name `FunctionModel` produces. That points at tests that run the **legacy tool loop** with `record_usage=True`, because only that path calls `record_run` (`_finish` records only when `runs` is non-empty, and the pipeline leaves `runs` empty).
- **`request.md` gaps:** the redirect mechanism is left to this plan. Whatever is chosen, refusal item 1 must hold. That rules out a silent global redirect on its own, because then a test that "forgot" the temporary directory would never be stopped.

## Current implementation

- `config/settings.py`: `USAGE_LOG_DIR = PROJECT_ROOT / "logs" / "usage"`.
- `harness/usage.py`:
  - `append_jsonl(records, log_dir=None)` falls back to `settings.USAGE_LOG_DIR`, creates the directory and appends to it. It is the **only** writer in the module.
  - `record_run(..., log_dir=None)` calls `append_jsonl` by its module-global name, inside `try: ... except Exception` that logs and returns `None`.
- `harness/agent.py`: `_finish` calls `usage.record_run(runs[-1], ...)` with **no** `log_dir` when `record_usage` is true and `runs` is non-empty. That only happens on the legacy `_ask_legacy` path. `_ask_pipeline` never records, because `runs` is always empty.
- `tests/conftest.py` only puts the repo root on `sys.path`. There is no guard and no shared redirect.

Tests I expect to write to the real directory today (from reading the code; the review counted records but did not name tests):

- **`tests/unit/test_agent_loop.py`:**
  - The `run()` helper calls `ask(agent, deps, question)` with the default `record_usage=True`. About 20 tests use it.
  - `test_a_replayed_answer_carries_the_envelope_the_tool_returned` and `test_truncation_does_not_narrow_what_grounding_accepts` also call `ask` directly with the default.
  - Tests that end in a raised exception (for example `UnexpectedModelBehavior`) never reach `_finish`, so they do not write.
  - `test_a_budget_trip_still_records_its_usage` (line 373) already redirects to `tmp_path`.
- **`tests/unit/test_run_cli.py`:** `test_one_clarification_round_trip_answers_the_question`, `test_a_non_interactive_caller_gets_the_clarification_not_a_guess`, `test_an_empty_reply_ends_the_conversation_rather_than_guessing` and `test_zero_clarification_rounds_still_runs_the_question_once` call `cli.converse(agent, ...)` with the default `record_usage=True`. `answered()` already passes `False`, and the pipeline tests do not record.
- **`tests/unit/test_live_tier.py`:** the default and `end-to-end` paths of `live.run_trial` call `ask(agent, deps, phrasing)` with the default. With `scripted_factory` this applies to:
  - `test_a_correct_run_grades_as_a_pass_through_the_real_runner`, `test_a_wrong_decision_grades_as_a_failure_and_says_which_grader`, and possibly `test_a_run_that_breaks_is_distinguishable_from_a_run_that_is_wrong`
  - `test_run_suite_reports_every_trial_and_writes_readable_transcripts` and `test_chunked_suite_resumes_without_repeating_completed_trials`
  - `test_stage_none_still_uses_the_legacy_tool_loop` and the three `test_stage_end_to_end_*` tests
- **Not writers:**
  - `test_demo.py`: replay mode passes `record_usage=False`.
  - `evals/replay.py`: passes `False`.
  - `test_intent_boundary.py` and `test_planner.py`: pipeline or planner only.
  - `test_usage.py`: uses `tmp_path` throughout.
- **`tests/live/test_live_eval.py`** (excluded by `-m "not live"`): its paid runs call `ask(build_agent(), ...)` and `live.run_trial` with the default, so they write when someone runs them by hand.

This list is an inference. The guard decides: once it is in place, any test I missed fails and names the directory.

## Proposed changes

1. **`tests/conftest.py`: the guard and the shared redirect fixture.**
   - At import, record the protected directories: `settings.USAGE_LOG_DIR.resolve()` captured before any test can monkeypatch it, plus `(PROJECT_ROOT / "logs" / "usage").resolve()` computed separately. Keep both in a module constant.
   - A **session-scoped autouse fixture** (`_usage_log_guard`) uses `pytest.MonkeyPatch.context()` to replace `harness.usage.append_jsonl` with a wrapper that has the same signature and does this:
     - Resolve the target as `(log_dir if log_dir is not None else settings.USAGE_LOG_DIR).resolve()`, reading `settings` at call time.
     - If the target equals a protected directory or is inside one (`Path.is_relative_to`), call `pytest.fail(...)` **before** writing. The message names the resolved target and the protected directory, cites ADR-021, and names the fix: "use the `usage_log_dir` fixture or pass `log_dir=tmp_path`".
     - Otherwise delegate to the original function.
   - Why `pytest.fail`: it raises `Failed`, a `BaseException` subclass (through `OutcomeException`). That passes through `record_run`'s `except Exception`, `live.run_trial`'s `except Exception`, `demo.run_step`'s `except Exception`, and `asyncio.run`, so the calling test fails rather than the error being swallowed.
   - A **non-autouse fixture `usage_log_dir`** monkeypatches `settings.USAGE_LOG_DIR` to `tmp_path / "usage"` and returns that path. This is the one shared way for a test to send the recorder to a temporary directory.
   - Leave the existing `sys.path` logic as it is.
2. **`tests/unit/test_usage_log_guard.py` (new): the demonstration required by refusal items 1 and 2.**
   - `test_recording_without_a_temporary_directory_fails_and_names_the_directory`:
     - Build a real run result the same way `test_record_run_against_real_result` does (`Agent(TestModel(...))`) and take a read-only snapshot of the real directory: file names and line counts, or empty if the directory does not exist.
     - Assert that `pytest.raises(pytest.fail.Exception)` fires around `usage.record_run(result, question="usage-guard demonstration")` with no `log_dir`.
     - Assert that the exception text contains the real directory's path.
     - Assert the snapshot is unchanged.
     - This goes through `record_run`'s never-raise wrapper. On the base commit nothing is raised, so the test fails with "DID NOT RAISE".
   - `test_append_without_a_directory_is_stopped_before_writing`: same check, calling `usage.append_jsonl` directly with no `log_dir`.
   - `test_another_spelling_of_the_real_directory_is_still_stopped`: pass `log_dir=REAL / "x" / ".."` and also a subdirectory of the real directory.
   - Positive controls, so the guard is not a blanket block:
     - an explicit `log_dir=tmp_path` writes and returns a path under `tmp_path`
     - with the `usage_log_dir` fixture, `record_run` with no `log_dir` writes into the fixture's directory
3. **`tests/unit/test_agent_loop.py`: opt into the redirect.** Add `pytestmark = pytest.mark.usefixtures("usage_log_dir")` at module level. Recording is still exercised, now against a temporary directory. Leave the line-373 test unchanged: its own `monkeypatch` to `tmp_path` applies after the module fixture's and is also temporary.
4. **`tests/unit/test_run_cli.py`:** add the same module-level `usefixtures("usage_log_dir")` marker.
5. **`tests/unit/test_live_tier.py`:** add the same module-level marker. The PF-4 subprocess `--collect-only` test is unaffected, because collecting tests does not run them.
6. **`tests/live/test_live_eval.py`:** add the same module-level marker so the manual paid tier is not stopped by the guard. This follows ADR-021 literally. The consequence is in the GAPs.
7. **`releases/_next/tests-never-write-usage-log.md` (new):** required by the delivery contract (`.method.yaml` `delivery`, `releases/README.md`, `pr-checks.yml`). Contents:
   - `**Type:** patch`, since only `tests/**` and `releases/**` change
   - `## Local deployment`: "Nothing to do."
   - `## Manual steps`: "Nothing to do."
   - `## Verification`: a `bash` block with the line-count before/after command below
   - `## Rollback`

Enforcing checks reviewed. None constrains the symbols this plan adds:

- `test_graders.py`'s meta-test covers only `evals.graders.grade_*`.
- `test_demo.py`'s source checks cover only `demo.py`.
- There is no `__all__` check or registry over test fixtures.
- `test_contract_pin.py` and `PINNED.json` cover only `contract/`.

So no enforcing file needs to gain an entry.

Rejected alternatives:

- **A global autouse redirect on its own.** Refusal item 1 could then never fire.
- **A teardown-only check.** It would report as an *error* at teardown, not a failure, and it would let the write happen first.
- **Changing `harness/usage.py` or `config/settings.py`.** That is product behaviour, which is out of scope, and it would raise the risk floor to `usage-log`.
- **A whole-session snapshot comparison of `logs/usage/`.** A real product run happening at the same time would make it fail falsely. The acceptance command below covers that check outside the suite.

## Verification approach

1. **Guard and demonstration:** run `.venv/bin/python -m pytest tests/unit/test_usage_log_guard.py -q`. All tests should pass.
2. **Whole suite (acceptance item 1):** run the suite between two counts and compare.
   ```bash
   (find logs/usage -type f -exec wc -l {} + 2>/dev/null; ls logs/usage 2>/dev/null) > /tmp/usage_before.txt
   .venv/bin/python -m pytest tests/ -q
   (find logs/usage -type f -exec wc -l {} + 2>/dev/null; ls logs/usage 2>/dev/null) > /tmp/usage_after.txt
   diff /tmp/usage_before.txt /tmp/usage_after.txt && echo UNCHANGED
   ```
   Expected: the suite passes with the same pass, skip and xfail counts as base plus the new tests, and the command prints `UNCHANGED`.

   If the suite crosses UTC midnight, `append_jsonl` would create a new date file. The comparison catches that too.
3. **Fails on base (refusal item 2), without touching the real `logs/usage/`:** extract the base tree into a throwaway directory, add only the new test file, and run it with the real venv. The copy's `PROJECT_ROOT`, and so its `USAGE_LOG_DIR`, is inside the temporary directory, so the base run's write lands there.
   ```bash
   T=$(mktemp -d); git archive 6d3dd93244d7bc8b87f2bb66bf2e0dd20b03b159 | tar -x -C "$T"
   cp tests/unit/test_usage_log_guard.py "$T/tests/unit/"
   (cd "$T" && "$OLDPWD/.venv/bin/python" -m pytest tests/unit/test_usage_log_guard.py -q); echo "exit=$?"
   ```
   Expected: the guard tests fail with `DID NOT RAISE` and the command prints `exit=1`. The positive controls pass on base too, which is fine; only the guard tests must fail. Throw the directory away afterwards. It is outside the repositories.
4. **Completeness of the enumeration:** in the full-suite run from step 2, a failure message naming `logs/usage` means a test was missed. Fix it by opting that module into `usage_log_dir`, inside the declared `tests/**` paths, never by weakening the guard.
5. **Recording still exercised (acceptance item 2):** `test_usage.py` (all of it) and `test_agent_loop.py::test_a_budget_trip_still_records_its_usage` pass and still assert records in a temporary directory.
6. **Live tier:** `tests/live` is not run (it needs credentials and spends tokens). Run `.venv/bin/python -m pytest tests/live --collect-only -q -m live` to confirm it still collects.

## Risks, assumptions and gaps

- **Assumption:** nothing between `record_run` and the test catches `BaseException`. That holds for `harness/agent.py`, `evals/live.py`, `demo.py`, `run.py` and asyncio's task machinery on CPython 3.10–3.13. If some layer did, the snapshot assertion in the demonstration test and acceptance step 2 would still show it.
- **Assumption:** every write to `logs/usage/` goes through `harness.usage.append_jsonl`. That is true at the base commit. A future writer that bypasses it would not be caught inside the suite, only by the acceptance command.
- **Risk:** a test that monkeypatches `harness.usage.append_jsonl` itself replaces the guard for its own duration. None does today.
- **Risk:** once ADR-021's pipeline recording is built, the pipeline tests that run with `record_usage=True` (`test_intent_boundary.py`, the pipeline tests in `test_run_cli.py`) will hit the guard. `test_run_cli.py` is already redirected; `test_intent_boundary.py` is not. That item will need to opt modules in. The guard failing there is the intended signal.
- Python 3.9+ `Path.is_relative_to` is available; `pr-checks.yml` uses 3.12.

## Planning gaps

GAP: request.md ("which tests write to the real directory") vs 05_measurements.md (counts records, names no tests). The writers listed in this plan are inferred from reading the code. The guard is the authoritative detector, and the executor may only add `usage_log_dir` opt-ins within `tests/**`.
GAP: docs/decisions.md ADR-021 ("Tests never write to `logs/usage/`") vs ADR-005/ADR-015 (usage recorded on every branch of a real run) and tests/live/test_live_eval.py. The paid live smoke tests will stop leaving usage records in `logs/usage/` once redirected. Whether paid live-test runs should be exempt is an owner decision. The plan follows the rank-1 rule literally.
GAP: request.md refusal item 2 ("must fail on the base commit") vs request.md ("files already polluted must not be cleaned") and ADR-021. Running the demonstration in place on the base commit would add a record to the real directory, so the plan runs the base check in an extracted copy of the base tree.
GAP: request.md ("already-polluted files") vs 05_measurements.md (2026-09-08 onwards contain only test records). Existing contamination stays in place and still distorts any usage report. Cleaning it is out of scope and needs a separate decision.
GAP: request.md ("tests may also write to `logs/harness.log`") vs ADR-021 (scoped to `logs/usage/`). Not checked and not in scope. `test_run_cli.py` already redirects `PROJECT_ROOT` for its probe log, but other tests are unchecked.
GAP: docs/decisions.md ADR-021 (pipeline must write request and run-summary records) vs harness/agent.py `_ask_pipeline` (writes none). Out of scope here. When it is implemented, pipeline tests will hit this guard and must opt into `usage_log_dir`.
GAP: request.md (names no release note) vs .method.yaml `delivery` / releases/README.md / .github/workflows/pr-checks.yml (every PR adds one `releases/_next/*.md`). Included as a delivery obligation, typed `patch`.

## Size estimate

Small and limited to the test suite: one conftest change, one new test module, module-level opt-in markers in four existing test files, and one release-note file. No harness, config, eval or contract code is touched, so no `risk_floor` interface or data entry is reached. The new files take the `new_files` floor of `normal`; everything else is `low`.

```yaml
size_estimate:
  files: 7
  scale: small
  subsystems: [tests, releases]
  paths:
    - "tests/conftest.py"
    - "tests/unit/test_usage_log_guard.py"
    - "tests/unit/test_agent_loop.py"
    - "tests/unit/test_run_cli.py"
    - "tests/unit/test_live_tier.py"
    - "tests/live/test_live_eval.py"
    - "releases/_next/tests-never-write-usage-log.md"
  interfaces: []
  data: []
```
