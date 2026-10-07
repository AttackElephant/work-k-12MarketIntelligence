<!-- Derived view of 001-contract.yaml. The YAML is authoritative. Regenerate; never edit. sha256=a7b78e6f3b44c4089049c1ffa59c5086ae4cebfdd22c248fb501b7153939067c -->

# Acceptance contract — work item 20261007T114835

contract_version: 0.1.0  ·  artifact_version: 1

## Observable behaviour
- **suite-leaves-usage-log-unchanged** — Count the lines in every file under `logs/usage/`, run `.venv/bin/python -m pytest tests/ -q`, then count again. The counts are the same and no new file has appeared in `logs/usage/`.
- **guard-stops-unredirected-recording** — While the test suite runs, any call that would make `harness.usage.append_jsonl` write to the real usage directory is stopped before anything is written. This covers a call through `record_run` with no `log_dir`, a direct call with no `log_dir`, and a call with an explicit `log_dir` that resolves to the real directory or to a folder inside it. The calling test fails, and the failure message names the directory it tried to write to.
- **guard-demonstrated-by-purpose-written-test** — A new test module, `tests/unit/test_usage_log_guard.py`, shows the guard stopping such a write. Its guard tests fail on the base commit 6d3dd93 with `DID NOT RAISE` and pass on the candidate.
- **recording-still-exercised** — Tests that exercise usage recording still record, now into a temporary directory, and still pass. This covers `tests/unit/test_usage.py`, `test_a_budget_trip_still_records_its_usage`, and the legacy tool-loop tests that run with `record_usage=True`. The shared non-autouse `usage_log_dir` fixture in `tests/conftest.py` sends the recorder to `tmp_path / "usage"` and returns that path.
- **guard-not-a-blanket-block** — A call that passes an explicit temporary `log_dir`, or that runs under the `usage_log_dir` fixture, writes normally. It returns a path inside the temporary directory.

## Required outputs
- `tests/conftest.py` — The plan's change 1: a session-scoped autouse guard around `harness.usage.append_jsonl`, plus the shared `usage_log_dir` fixture. The existing `sys.path` logic stays as it is.
- `tests/unit/test_usage_log_guard.py` — The plan's change 2: the purpose-written test that request refusal items 1 and 2 call for, with positive controls.
- `releases/_next/tests-never-write-usage-log.md` — Delivery contract (`.method.yaml` delivery, `releases/README.md`, `pr-checks.yml`): every PR adds exactly one release note. It is typed patch.

## Deterministic criteria
- **guard-module-passes** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_usage_log_guard.py -q`
  - expected: Fails on the base commit because `tests/unit/test_usage_log_guard.py` does not exist there: pytest exits 4 with 'file or directory not found'. Passes once the guard is in `tests/conftest.py` and the module exists: every guard test and positive control passes.
  - machine-checked: yes
- **guard-test-fails-on-base-tree** in `k-12Harness`
  - command: `bash -c 'R="$(pwd)"; T="$(mktemp -d)"; git archive 6d3dd93244d7bc8b87f2bb66bf2e0dd20b03b159 | tar -x -C "$T"; if cp tests/unit/test_usage_log_guard.py "$T/tests/unit/"; then (cd "$T" && "$R/.venv/bin/python" -m pytest tests/unit/test_usage_log_guard.py -q -p no:cacheprovider); echo "base_exit=$?"; else echo "base_exit=missing-test"; fi; rm -rf "$T"'`
  - expected: Runs the candidate's guard test module against an extracted copy of the base tree. The copy's own `logs/usage/` is inside the temporary directory, so the real directory is never touched. On the base commit it fails because the test module does not exist yet: the command prints `base_exit=missing-test`. On the candidate the guard tests fail in the base copy with `DID NOT RAISE`, because base code has no guard and `record_run` swallows any `Exception`. The command then prints `base_exit=1`. This shows that the demonstration measures the change (request refusal item 2).
  - machine-checked: yes
- **usage-log-unchanged-by-suite** in `k-12Harness`
  - command: `bash -c 'snap(){ if [ -d logs/usage ]; then find logs/usage -type f -exec wc -l {} \; | sort; ls -1 logs/usage; fi; }; B="$(snap)"; .venv/bin/python -m pytest tests/ -q >/dev/null 2>&1; rc=$?; A="$(snap)"; if [ "$B" = "$A" ]; then echo "suite_exit=$rc usage_log=UNCHANGED"; else echo "suite_exit=$rc usage_log=CHANGED"; fi'`
  - expected: This is request item 1, checked against the real directory. It fails on the base commit because the legacy tool-loop tests in test_agent_loop.py, test_run_cli.py and test_live_tier.py append `function:*` records to `logs/usage/usage-<UTC date>.jsonl`, so the command prints `usage_log=CHANGED`. It passes once those tests use `usage_log_dir` and the guard stops any other writer: the suite exits 0 and the file names and line counts are identical before and after. See decision-base-dry-run-writes.
  - machine-checked: yes
- **release-note-present-and-shaped** in `k-12Harness`
  - command: `.venv/bin/python -c "import pathlib,re; t=pathlib.Path('releases/_next/tests-never-write-usage-log.md').read_text(); v=t.split('## Verification',1)[1].split('\n## ',1)[0]; ok=all([re.search(r'(?m)^\*\*Type:\*\* *patch *$',t), re.search(r'(?m)^## Local deployment',t), re.search(r'(?m)^## Manual steps',t), re.search(r'(?m)^## Rollback',t), re.search(r'(?m)^\x60{3}bash',v)]); print('NOTES_OK' if ok else 'NOTES_BAD'); raise SystemExit(0 if ok else 1)"`
  - expected: Fails on the base commit because `releases/_next/tests-never-write-usage-log.md` does not exist (FileNotFoundError). Passes once the note exists with a `**Type:** patch` line and the `## Local deployment`, `## Manual steps`, `## Verification` (with at least one fenced bash block) and `## Rollback` sections. The command prints NOTES_OK.
  - machine-checked: yes
- **full-test-suite** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/ -q`
  - expected: The repository's declared verify command exits zero with no failures, errors or unexpected passes. The existing strict xfail (O-8) stays xfailed. Pass, skip and xfail counts equal the base counts plus the new guard tests.
  - machine-checked: yes
- **recording-tests-still-pass** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_usage.py "tests/unit/test_agent_loop.py::test_a_budget_trip_still_records_its_usage" -q`
  - expected: The tests that assert usage records in a temporary directory pass before and after the change (request item 2).
  - machine-checked: yes
- **live-tier-still-collects** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/live --collect-only -q -m live`
  - expected: The manual live tier still collects under `-m live`. Nothing is run and no tokens are spent.
  - machine-checked: yes
- **no-product-code-changed** in `k-12Harness`
  - command: `git diff --quiet 6d3dd93244d7bc8b87f2bb66bf2e0dd20b03b159 -- harness config evals prompts contract server sql pytest.ini logging_config.yaml requirements.txt run.py demo.py capture.py authcheck.py VERSION .method.yaml .github .claude`
  - expected: Exits 0: no product, configuration, evaluation, contract, entry-point, gate, VERSION or method-declaration file differs from the base commit. Exits 1 if any does.
  - machine-checked: yes

## Must not change
- `harness/usage.py`: the signatures and product behaviour of `append_jsonl` and `record_run`, including the `settings.USAGE_LOG_DIR` fallback and the rule that `record_run` never raises (ADR-005).
- `config/settings.py`: `USAGE_LOG_DIR = PROJECT_ROOT / "logs" / "usage"`.
- The usage record format: `LlmRequestRecord`, `RunSummaryRecord` and `SCHEMA_VERSION`.
- What the product records in normal use, outside the test suite.
- Existing files under `logs/usage/`. They are not deleted, truncated or edited by the executor or by any test.
- `pytest.ini` and its `addopts = -m "not live"`.
- `VERSION`. No `releases/v*.md` file is added.
- The assertions of existing tests. No existing test is deleted, skipped, xfailed or rewritten to get around the guard. Edits to existing test modules are limited to adding the `usage_log_dir` opt-in marker.
- The existing `sys.path` logic in `tests/conftest.py`.

## Failure behaviour
- A test calls `usage.record_run(...)` with no `log_dir`, and `settings.USAGE_LOG_DIR` still points at the real `logs/usage/`. → The guard calls `pytest.fail` before anything is written. `Failed` is a `BaseException`, so it passes through `record_run`'s `except Exception`. The test is reported as failed, not as an error, and the failure is not swallowed. The message names the resolved target directory and the protected directory, cites ADR-021, and names the fix: use the `usage_log_dir` fixture or pass `log_dir=tmp_path`. No file under `logs/usage/` is created or grows.
- A test calls `usage.append_jsonl(...)` directly with no `log_dir` while the real directory is in effect. → The same failure, raised before the directory is created or a file is opened.
- A test passes an explicit `log_dir` that resolves to the real directory or to a folder inside it, such as `REAL / "x" / ".."` or a subdirectory. → The same failure. The target is resolved before it is compared with the protected directories.
- A test monkeypatches `settings.USAGE_LOG_DIR` back to the real directory. → It is still stopped. The protected directories were captured when `tests/conftest.py` was imported, before any test ran, and the target is read from `settings` at call time.
- A legacy tool-loop run goes through `ask`/`converse`/`live.run_trial` with `record_usage=True` in a module that has not opted into `usage_log_dir`. → The test fails and names the real directory. The fix is to add the opt-in marker to that module, inside the permitted files. The guard is never weakened.

## Compatibility
- ADR-021: 'Tests never write to `logs/usage/`.' This contract enforces only this sentence. The pipeline request and run-summary records are not part of it.
- ADR-005: `record_run()` never raises in product use. The guard exists only inside the test session (`tests/conftest.py`), and product code is unchanged.
- ADR-007: the live tier stays manual and excluded by default (`-m "not live"`), never a PR gate.
- Delivery contract: exactly one `releases/_next/*.md` is added, `VERSION` is untouched, and the note is typed patch because only `tests/**` and `releases/**` change (`.method.yaml` release classes).
- The supported interpreters are CPython 3.10 to 3.13 (`pr-checks.yml` uses 3.12). `Path.is_relative_to` is available on all of them.

## Compatibility baselines
_none_

## Contract requirements
- **scope-is-the-file-list** — The permitted change set is exactly `required_outputs` plus `permitted_files` in k-12Harness. The work-store is never written by the executor. A change to the contingent test modules is allowed only when the guard shows that module writing to the real directory, and the change is limited to adding the `usage_log_dir` opt-in marker.
- **guard-mechanism** — The guard wraps `harness.usage.append_jsonl` for the whole session, with the same signature, and fails through `pytest.fail` before any write. It does not use a teardown check, a silent redirect on its own, or a session-wide directory snapshot (plan, rejected alternatives).
- **decision-base-dry-run-writes** — DECISION: may the controller's dry run against base commit 6d3dd93 execute the base test suite in place? The base code has no guard, so `full-test-suite` and `usage-log-unchanged-by-suite`, run on base, append scripted-model records to the real `logs/usage/`. The plan avoided exactly that for its own base check. Options: (a) accept it: the directory is untracked, already contaminated, and cleaning it is out of scope; the contract stands as written. (b) no base run may write there: both criteria run only on the candidate, and a human records the base result as accepted; or they are rewritten to run the suite in an isolated copy of the tree (as `guard-test-fails-on-base-tree` does), with the base failure shown by a copy-based count of the copy's own `logs/usage/`. Affects: deterministic `full-test-suite` and `usage-log-unchanged-by-suite`, and observable_behaviour suite-leaves-usage-log-unchanged. This contract is written on option (a), because the repository's declared verify command must be run and the plan's verification step 2 runs the suite in place.
- **decision-live-tier-usage** — DECISION: should paid live-tier runs (`tests/live/test_live_eval.py`, run by hand with `-m live`) still record usage in the real `logs/usage/`? Options: (a) no: follow ADR-021 literally and redirect the module with `usage_log_dir` (the plan's choice); real-model live-test telemetry then no longer lands in `logs/usage/`. (b) yes: exempt live-marked tests from the guard, which needs an exemption rule in `tests/conftest.py`, leaves `tests/live/test_live_eval.py` unchanged, and departs from ADR-021's wording. Affects: permitted_files `tests/live/test_live_eval.py`, required_outputs `tests/conftest.py` (exemption logic), compatibility ADR-021, and semantic `live-tier-consistent`. This contract is written on option (a).
- **assumption-baseexception-passes** — ASSUMPTION: no layer between `append_jsonl` and the test catches `BaseException` (`harness/agent.py`, `evals/live.py`, `demo.py`, `run.py`, asyncio). If one does, the snapshot assertion in the demonstration test and `usage-log-unchanged-by-suite` still show it.
- **assumption-single-writer** — ASSUMPTION: every write to `logs/usage/` goes through `harness.usage.append_jsonl`, as it does at the base commit. A writer that bypasses it is caught only by `usage-log-unchanged-by-suite`.
- **gap-writers-inferred** — GAP: the plan's list of writing tests is inferred from reading the code. The guard is the authoritative detector. Writers it finds are fixed by opt-in inside the permitted files. If one turns up outside them, that is recorded as a scope defect.
- **gap-polluted-files** — GAP: the files already contaminated in `logs/usage/` stay in place and still distort usage reports. Cleaning them needs a separate decision.

## Semantic criteria
- **guard-fails-before-write** — The guard decides and calls `pytest.fail` inside the wrapper, before calling the original `append_jsonl`. So the directory is not created and no file is opened for a protected target, and the outcome is a test failure, not a teardown error.
- **message-names-directory** — The failure message contains the resolved path the call tried to write to and the protected directory, cites ADR-021, and names the `usage_log_dir` fixture or `log_dir=tmp_path` as the fix. The demonstration test asserts that the real directory's path appears in the exception text.
- **demonstration-is-load-bearing** — `tests/unit/test_usage_log_guard.py` calls `record_run` with a real run result and no `log_dir`, and also calls `append_jsonl` directly. It asserts the failure with `pytest.raises(pytest.fail.Exception)` and asserts that a read-only snapshot of the real directory is unchanged. It would fail if the guard were removed; `guard-test-fails-on-base-tree` shows this. The test never writes to the real directory, even when it fails.
- **protected-dirs-captured-early** — The protected directories are captured when `tests/conftest.py` is imported. They are `settings.USAGE_LOG_DIR.resolve()` and a separately computed `(PROJECT_ROOT / "logs" / "usage").resolve()`, so a test's monkeypatch cannot move them. The target is resolved at call time from `log_dir` or the current `settings.USAGE_LOG_DIR`.
- **positive-controls-present** — The guard module includes positive controls. An explicit `log_dir=tmp_path` writes and returns a path under `tmp_path`, and under `usage_log_dir` a `record_run` with no `log_dir` writes into the fixture's directory. So the guard is shown not to be a blanket block.
- **opt-ins-preserve-coverage** — Each opt-in is a module-level `pytest.mark.usefixtures("usage_log_dir")`, or the fixture itself. Recording is still exercised: no test switches to `record_usage=False`, and no test is skipped, removed or loosened to avoid the guard.
- **live-tier-consistent** — Under decision-live-tier-usage option (a), `tests/live/test_live_eval.py` opts into `usage_log_dir`, so a manual paid run is not stopped by the guard and does not write to the real directory.
- **release-note-accurate** — The release note states that only tests and release notes changed, says 'Nothing to do.' for local deployment and manual steps, and gives a pasteable Verification bash block (no comment lines) that compares the line counts in `logs/usage/` before and after the suite, with its expected output stated in prose.

## Out of scope
- Deleting, cleaning or rewriting the files already in `logs/usage/`.
- Whether tests write to `logs/harness.log`.
- ADR-021's pipeline request and run-summary records, the `schema_version` bump, and any change to what the product records in normal use.
- Any change to `harness/usage.py`, `config/settings.py` or the usage record format.
- Running the paid live tier (`tests/live` under `-m live`) as part of verification.
- Opting pipeline-only test modules into `usage_log_dir` ahead of the future ADR-021 pipeline-recording work, unless the guard shows them writing today.

## Base-commit dry run

Every deterministic criterion run once at `k-12Harness` @ 6d3dd93244d7, before any change (2026-10-07T12:14:58Z). A flag means the result cannot depend on the change.

- **guard-module-passes** (behaviour) — ran, exit 4 on the base, expectation not met
- **guard-test-fails-on-base-tree** (behaviour) — FLAGGED, exit 0 on the base, expectation not met
  - excludes-matches-base: stdout_excludes 'base_exit=(0|missing-test)' already matches the base commit's output on the line 'base_exit=missing-test', while the exit code met the expectation; no candidate can pass unless that line disappears — **unaccepted**: fix the criterion in a new contract version, or accept it
- **usage-log-unchanged-by-suite** (behaviour) — ran, exit 0 on the base, expectation not met
- **release-note-present-and-shaped** (behaviour) — ran, exit 1 on the base, expectation not met
- **full-test-suite** (regression) — ran, exit 0 on the base, expectation met
- **recording-tests-still-pass** (regression) — ran, exit 0 on the base, expectation met
- **live-tier-still-collects** (regression) — ran, exit 0 on the base, expectation met
- **no-product-code-changed** (regression) — ran, exit 0 on the base, expectation met
