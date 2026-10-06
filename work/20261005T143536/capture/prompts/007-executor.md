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

Work item: 20261005T143536
Request: # Request

## Objective

When `run.py` prints an outcome as text, standard output holds the outcome and nothing else, and the confidence block shows the data layer's grade as its value and weakest input, with each component's grade where the response carries them, instead of a printed Python dictionary.

## Target repository

`k-12Harness`

## What should happen

1. Render the saved real response `docs/reviews/2026-10-poc-review/_work/envelopes/L01.json` (question VC-POC-01, Camberwell Grammar School) as an answer. The confidence block shows the grade "Observed" and its weakest input "Observed", read from `confidence.grade.value` and `confidence.grade.weakest_input`. It still shows the four ratings as the response states them: forecast `not_applicable`, linkage `high`, source `high`, temporal `medium`. No line of the block contains `{` or `'adjustments'`.
2. Render `docs/reviews/2026-10-poc-review/_work/envelopes/direct_VC-POC-13_579499.json` (VC-POC-13). Its `confidence.grade.component_grades` names two components, `sa2_age_projection` and `sa2_projection`, each "Indicative". The block shows each component with its own grade (architecture §6, rule 5).
3. Run the command line in its default text mode. No line of standard output is a log record (a JSON object with a `"level"` key). The log file `logs/harness.log` still receives its records.

## What should be refused or fail

1. A response whose `confidence` has no `grade`, or a `grade` with no `value`: the block shows the ratings only and says no grade was stated. The harness does not supply a grade of its own (ADR-009).
2. `--json` mode: standard output is exactly one JSON document, the same as before this change. A test that parses it fails if anything else is printed.
3. The existing logging tests in `tests/unit/test_logging_config.py` still pass unchanged: `harness.*` loggers follow the single `LOG_LEVEL` setting and the transport libraries stay quiet (ADR-016, finding 3).

## Known acceptance inputs

- The two saved responses above, read by tests from the review folder where they are filed.
- `run.py`, function `render_text` (the confidence lines are near line 105 at commit `e4bd76b`).
- `tests/unit/test_run_cli.py`, `test_confidence_renders_as_the_axes_the_data_layer_stated` (line 72): it asserts that the block's keys equal the keys of `confidence` "with no overall grade added". That was the rule before the contract carried a grade. It must change to match ADR-009's rule for a stated grade, without allowing a grade the response did not state.
- `tests/unit/test_demo.py` asserts only that the heading "Confidence (as stated by the data layer):" is present, so the demo's tests do not depend on the block's content.
- `logging_config.yaml` (the console handler writes to `ext://sys.stdout`) and `config/settings.py` (`LOG_LEVEL` defaults to `DEBUG`).
- The grade's shape, including `component_grades`: `contract/poc_evidence_contract.v0.3.0.schema.json` (line 404).

## Gaps not yet settled

- GAP: where console log records go. I found no position on it in the decisions log, the architecture document, the runbook or the logging configuration, beyond the rule that one setting governs the level. Three ways meet the objective: send the console handler to standard error; switch console logging off for the command line; or change the default level. Changing the default level also changes what the log file records.
- GAP: the shape of `weakest_input` was seen in one response (`{"grade": "Observed"}`). Other responses may carry more keys.
- GAP: a `Modelled` grade must show its backtest block (ADR-009). The only Modelled question is gated and never rendered today. Out of scope here; say so in the contract.

## Out of scope

Any change to what is answered, to the JSON output, to the log file's content, or to the rendering of anything except the confidence block.
Component repository: /Users/shane/Projects/k-12Harness
Current contract: /Users/shane/Projects/work/k-12MarketIntelligence/work/20261005T143536/acceptance/002-contract.yaml
Plan: /Users/shane/Projects/work/k-12MarketIntelligence/work/20261005T143536/planning/001-plan.md

Declared editable files:
- logging_config.yaml
- releases/_next/cli-confidence-grade-and-clean-stdout.md
- run.py
- tests/unit/test_run_cli.py

Read-only context supplied to the executor transport:
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20261005T143536/acceptance/002-contract.yaml
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20261005T143536/planning/001-plan.md
- /Users/shane/Projects/k-12Harness/contract/poc_evidence_contract.v0.3.0.schema.json
- /Users/shane/Projects/k-12Harness/docs/architecture.md
- /Users/shane/Projects/k-12Harness/docs/decisions.md
- /Users/shane/Projects/k-12Harness/docs/reviews/2026-10-poc-review/_work/envelopes/L01.json
- /Users/shane/Projects/k-12Harness/docs/reviews/2026-10-poc-review/_work/envelopes/direct_VC-POC-13_579499.json
