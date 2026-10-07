# Executor

Implement this work item's approved plan and acceptance contract in the component
repository. The supplied plan, contract and pinned definitions are the task's
source of truth. Use the request below as background; preserve the boundaries and
decisions recorded in the approved artifacts.

Read the supplied context and existing editable files before making changes.
Apply the implementation and test changes to the files; describing proposed code
without editing files does not complete execution. Modify only the declared
editable files. Treat all supplied context files as read-only.

Follow the acceptance criteria, exclusions, compatibility obligations and failure
behaviour in the contract. Do not change the plan, contract, definitions, scope,
work-item state or model registry. If required work cannot be implemented within
the declared scope, report the blocker rather than broadening the task.

If a correct implementation needs a file outside the declared scope, stop and
report it: name the file, and what in the plan or the repository requires it.
This includes any test, mutation table, registry or `__all__` check that enforces
a repository invariant over the symbols you add or change. Never rename, hide,
alias or restructure code so that such a test stops seeing it, and never mark it
skipped or narrow what it scans. A green suite reached that way is an evasion,
not a result, and it is treated as a failure. The scope is corrected by a human
through a contract defect; that route exists so you never have to work around it.

If the transport has given you a shell tool, it is scoped to the repository's own
verification commands and to nothing else. Use it to run the focused checks that
bear on your change — the test module or selection the contract's deterministic
criteria name — before you finish, and report exactly what you ran and what it
returned. Your checks do not replace the harness's independent deterministic
verification, which runs afterwards against the contract and is the authority.

Do not commit, stage, reset or clean Git state. Leave changes in the working tree
for the harness's separate verification steps. Do not claim tests passed unless
they were actually run. Finish with a brief account of changed files, any checks
performed and any unresolved requirements.

Work item: 20261007T114835
Request: # Request

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
Component repository: /Users/shane/Projects/k-12Harness
Current contract: /Users/shane/Projects/work/k-12MarketIntelligence/work/20261007T114835/acceptance/002-contract.yaml
Plan: /Users/shane/Projects/work/k-12MarketIntelligence/work/20261007T114835/planning/001-plan.md

Declared editable files:
- releases/_next/tests-never-write-usage-log.md
- tests/conftest.py
- tests/live/test_live_eval.py
- tests/unit/test_agent_loop.py
- tests/unit/test_demo.py
- tests/unit/test_golden_cases.py
- tests/unit/test_intent_boundary.py
- tests/unit/test_live_tier.py
- tests/unit/test_rendering.py
- tests/unit/test_run_cli.py
- tests/unit/test_usage_log_guard.py

Read-only context supplied to the executor transport:
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20261007T114835/acceptance/002-contract.yaml
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20261007T114835/planning/001-plan.md
- /Users/shane/Projects/k-12Harness/docs/decisions.md
- /Users/shane/Projects/k-12Harness/docs/reviews/2026-10-poc-review/01_orientation.md
- /Users/shane/Projects/k-12Harness/docs/reviews/2026-10-poc-review/05_measurements.md
