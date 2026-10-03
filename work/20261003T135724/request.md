# Request

# Request

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
