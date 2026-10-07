# Request

## Objective

Running the test suite never writes to the real usage log directory (`logs/usage/`), and a test that tries to is stopped.

## Target repository

`k-12Harness`

## What should happen

1. Count the lines in every file under `logs/usage/`, run `.venv/bin/python -m pytest tests/ -q`, and count again. The counts are equal, and no new file has appeared there.
2. Tests that exercise usage recording still do so, against a temporary directory, and still pass.

## What should be refused or fail

1. A test that calls the usage recorder without sending it to a temporary directory fails, and the failure names the directory it tried to write to. This must be demonstrated by a test written for the purpose, not assumed.
2. The check in (1) must fail on the code as it is before this change. If it passes on the base commit, the check is not measuring anything.

## Known acceptance inputs

- The rule: ADR-021 in `docs/decisions.md`, "Tests never write to `logs/usage/`".
- `config/settings.py`: `USAGE_LOG_DIR = PROJECT_ROOT / "logs" / "usage"`.
- `harness/usage.py`: `append_jsonl` and `record_run` fall back to `settings.USAGE_LOG_DIR` when no directory is given.
- The finding: `docs/reviews/2026-10-poc-review/05_measurements.md` ("`logs/usage/` is contaminated by the unit test suite: 2026-09-29 has 468 lines, all `function:*` test models") and `01_orientation.md`, conflict C2.
- `tests/unit/test_agent_loop.py` line 373 already redirects the directory for one test; `tests/unit/test_usage.py` passes a temporary directory explicitly.

## Gaps not yet settled

- GAP: which tests write to the real directory. The review counted the records and did not name the tests.
- GAP: how the redirect is done (one shared fixture for every test, or a guard that fails on a write) is for the plan. Either meets the objective. "What should be refused or fail", item 1, must hold whichever is chosen.
- GAP: the files already polluted. `logs/` is not tracked by git, so they exist only on this machine. Deleting or cleaning them is not part of this item and must not be done by an executor.
- GAP: tests may also write to `logs/harness.log`. Not checked, and not in scope.

## Out of scope

Any change to what the product records in normal use. Any change to the usage record format.
