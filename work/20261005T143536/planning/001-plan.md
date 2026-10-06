# Implementation Plan

## Bounded objective

This work changes two things that `run.py` writes to standard output in its default text mode:

1. **The confidence block in `run.render_answer`.**
   - It shows the data layer's grade, read from `confidence.grade.value`.
   - It shows the grade's weakest input, read from `confidence.grade.weakest_input.grade`.
   - Where the response carries `confidence.grade.component_grades`, it shows each component with its own grade.
   - The four axis ratings stay, as the response states them.
   - No line of the block is a printed Python dictionary.
   - When the response states no grade, or a grade with no `value`, the block shows the ratings and says that no grade was stated.
2. **Console log records.** They must stop reaching standard output. The log file `logs/harness.log` and the `LOG_LEVEL` behaviour do not change.

Nothing else changes:
- what is answered;
- the `--json` document;
- the log file's content;
- every other rendered block (prose, data status, signals, limitations, provenance, footer).

Rendering a `Modelled` grade with its backtest block is out of scope (see Planning gaps).

## Definition Pack interpretation

- **ADR-009 (rank 1).** Confidence renders exactly as the response states it. Before CR-4, that meant the axes only. Now that the response carries a grade, it means "the envelope's grade and weakest-input, verbatim". The harness never assigns a grade.
  - So every grade word in the block must be copied from the response.
  - When `grade` or `grade.value` is missing, the block says no grade was stated. It does not use a placeholder grade or one worked out from the axes.
- **Architecture §6 rule 5 (rank 2).** After CR-4, `grade.value` and `weakest_input` are the headline, and a compound answer shows each component's own grade. The rule also requires two things this work does not cover:
  - a `Modelled` figure renders with the `backtest` block that earned it;
  - `claim_eligibility` gates the wording.

  Both are outside this request: the request is about the confidence block only, and the only `Modelled` question is gated.
- **Contract schema `confidence.grade` (rank 3, line ~404).**
  - `value`, `current_ceiling`, `maximum_grade` and `rule_id` are required.
  - `weakest_input` is an object that requires `grade` and may carry a `reference` string. The schema does not forbid other keys in it.
  - `component_grades` is optional. Each entry requires `value` and `rule_id`.
  - `adjustments` and `backtest` are present too.

  Only `value`, `weakest_input`, `weakest_input.reference` (if present) and each component's `value` are rendered. The request asks for those, and they are what the schema declares.
- **ADR-016 finding 3 and `logging_config.yaml`.** `LOG_LEVEL` is the single verbosity setting for `harness.*` loggers, and `anthropic`, `httpx` and `httpcore` are pinned at WARNING. Neither may change. None of the documents in the pack say where console records go (see the first GAP).
- **ADR-017 decision 3.** The demo renders through `run.render_text`, so the demo's confidence block changes along with this one. `test_demo.py` checks only the heading, which stays word for word.
- **The two saved responses (rank 9).** They are evidence of real response shapes and serve as test inputs. They are not authority.

## Current implementation

- `run.py::render_answer` (around line 105) builds the block like this:

  ```python
  axes = [f"{axis}: {rating}" for axis, rating in sorted(output.confidence.items())]
  ```

  `output.confidence` is the envelope's `confidence` copied whole (`RenderedAnswer.from_envelope`), and since v0.3.0 it includes `grade`. So the block prints a line `grade: {'adjustments': [], 'backtest': {...}, ...}`. That is the printed Python dictionary the request describes. The docstring still says "carry no overall grade".
- `tests/unit/test_run_cli.py::test_confidence_renders_as_the_axes_the_data_layer_stated` (line 72) checks two things:
  - `f"{axis}: {rating}"` appears for every key. For `grade`, that means the dictionary repr.
  - The set of line labels equals `set(output.confidence)`.

  The stub fixture (`evals/build_stub_fixtures.py::_grade`) gives every envelope a `grade`, so this test currently passes only because the dictionary is printed.
- `logging_config.yaml`:
  - The `console` handler is a `StreamHandler` on `ext://sys.stdout` with the JSON formatter.
  - The root logger and `harness.retry`, `harness.agent`, `harness.usage` and `server` route to `console` and `file_main`.
  - `configure_logging()`, called first in `run.main`, sets the root level from `LOG_LEVEL`, which defaults to DEBUG.
  - Result: in text mode and in `--json` mode, JSON log records are mixed into standard output.
- `run.main` prints `output.model_dump_json(indent=2)` for `--json`, otherwise `render_text(output)`. Operational errors go to standard error.
- `tests/unit/test_logging_config.py` checks only logger levels and enabled states. It does not check handler streams.

## Proposed changes

1. **`run.py`**, rendering of the confidence block only:
   - Add a private helper, e.g. `_confidence_lines(confidence: dict) -> list[str]`, called from `render_answer`. The heading `Confidence (as stated by the data layer):` and the bullet format stay as they are.
   - **Axis lines.** For every key except `grade`, in sorted order, render `"{axis}: {rating}"` when the value is a string, number or boolean. This keeps the current lines for `forecast`, `linkage`, `source` and `temporal`.
     - If a non-`grade` value is a dict or list, it is not printed as a repr. The schema allows no such value (`additionalProperties: false`), so render it as `"{axis}: (not shown — unexpected structure)"`, or skip it with a code comment. The executor chooses; neither may print `{`.
   - **Grade lines.** Only when `confidence.get("grade")` is a dict with a non-empty string `value`:
     - `grade: <value>`
     - `weakest input: <weakest_input.grade>`, followed by ` (<reference>)` when `weakest_input.reference` is a string. When `weakest_input` or its `grade` is missing, `weakest input: not stated by the data layer`.
     - For each entry of `component_grades`, sorted by name: `component <name>: <value>`. When an entry has no `value`, `component <name>: not stated by the data layer`.
     - `adjustments`, `backtest`, `current_ceiling`, `maximum_grade` and `rule_id` are not rendered.
   - **No grade stated.** When `grade` is missing, is not a dict, or has no string `value`, render one line: `grade: not stated by the data layer`. No grade word appears.
   - Update the docstring of `render_answer` and the module docstring's ADR-009 sentence: the grade is shown when stated and never supplied.
   - No change to `converse`, `main`, the exit codes, `--json`, or any other block.
2. **`logging_config.yaml`**, console routing only (subject to the first GAP below):
   - Change `handlers.console.stream` from `ext://sys.stdout` to `ext://sys.stderr`.
   - Update the header comment to say that console records go to standard error, so that a command line's standard output carries only its outcome.
   - No change to levels, the formatter, `file_main`, the logger entries or the transport pins. `tests/unit/test_logging_config.py` therefore passes unchanged.
3. **`tests/unit/test_run_cli.py`**:
   - **Replace** `test_confidence_renders_as_the_axes_the_data_layer_stated` with a test matching ADR-009's rule for a stated grade:
     - every axis appears as `"{axis}: {rating}"`;
     - the `grade:` line equals `confidence["grade"]["value"]`;
     - the `weakest input:` line equals `confidence["grade"]["weakest_input"]["grade"]`;
     - every block label is in `{non-grade axes} ∪ {"grade", "weakest input"} ∪ {"component <name>" for each stated component}`. Nothing is added beyond what the response states.
     - Update the module docstring's "with no overall grade added".
   - **Add** a helper that builds a `RenderedAnswer` from a saved response with `RenderedAnswer.from_envelope(AnswerDraft(question_id=..., answer="…"), envelope)`. It reads `PROJECT_ROOT / "docs/reviews/2026-10-poc-review/_work/envelopes/<name>.json"`.
   - **Add** `test_l01_block_shows_the_stated_grade_and_weakest_input`:
     - lines `grade: Observed` and `weakest input: Observed`;
     - `forecast: not_applicable`, `linkage: high`, `source: high`, `temporal: medium`;
     - no block line contains `{` or `'adjustments'`.
   - **Add** `test_compound_answer_shows_each_component_grade` (VC-POC-13 response): lines `component sa2_age_projection: Indicative` and `component sa2_projection: Indicative`, and no `{`.
   - **Add** `test_no_stated_grade_shows_ratings_only_and_says_so`, parametrised over two cases: `grade` removed, and `grade.value` removed (on a deep copy of L01). For each:
     - the four ratings are present;
     - the "not stated" wording is present;
     - none of `Observed`, `Derived`, `Modelled`, `Indicative`, `Unsupported` appears in the block.
   - **Add** `test_text_mode_stdout_carries_no_log_record`:
     - Monkeypatch `settings.PROJECT_ROOT` to `tmp_path`, so the file handler writes to `tmp_path/logs/harness.log` and not to the repository's log.
     - Monkeypatch `require_credentials` and `build_client`, and replace `converse` with a fake that logs a probe record through `logging.getLogger("harness.agent")` and then returns an `Unsupported`.
     - Run `cli.main([...])` with the real `configure_logging`.
     - Check that standard output is exactly `cli.render_text(<the Unsupported the fake returned>)` followed by one newline, so it holds the outcome and nothing else; that no standard-output line parses as a JSON object with a `"level"` key; and that the probe message appears in `tmp_path/logs/harness.log`.
     - Afterwards, undo the monkeypatches and call `configure_logging()` again so later tests get clean handlers.
   - **Add** `test_json_mode_stdout_is_exactly_one_json_document`: the same setup with `--json`. `json.loads(capsys.readouterr().out)` succeeds, which means nothing else was printed, and it equals `output.model_dump(mode="json")`.
   - Existing tests that check enforcement elsewhere stay unchanged: `test_demo.py`, which checks the heading and that the demo has no renderer of its own, and `test_logging_config.py`. There is no `__all__`, mutation table or registry covering `run.py`'s symbols. The new helper is private, and `test_rendering.py` / `test_graders.py` cover `harness/rendering.py` and `evals/graders.py`, which this work does not touch.
4. **`releases/_next/cli-confidence-grade-and-clean-stdout.md`** (new): copied from `releases/_template.md`.
   - `**Type:** minor`, because `run.py` and `logging_config.yaml` are `minor_paths`.
   - Summary: the confidence block shows the stated grade, and console logs now go to standard error.
   - `## Local deployment`: nothing to do.
   - `## Manual steps`: nothing to do.
   - `## Verification`: a bash block running `.venv/bin/python -m pytest tests/unit/test_run_cli.py tests/unit/test_logging_config.py -q`.
   - `## Rollback`.
   - `VERSION` is not touched.

## Verification approach

- Deterministic: `.venv/bin/python -m pytest tests/ -q`, the component's verify command. In particular:
  - the new and rewritten tests in `tests/unit/test_run_cli.py`;
  - `tests/unit/test_logging_config.py` unchanged and passing (refused-or-fail case 3);
  - `tests/unit/test_demo.py` passing, including the rehearsal checks for the heading, limitations and provenance.
- Acceptance mapping:
  - **What should happen 1:** the L01 test.
  - **What should happen 2:** the VC-POC-13 test.
  - **What should happen 3:** the text-mode standard-output test, plus the log-file check against `tmp_path`.
  - **Refused 1:** the parametrised no-grade test.
  - **Refused 2:** the `--json` single-document test.
  - **Refused 3:** the unchanged logging tests.
- Read-only check before editing: grep the repository for `render_text`, `Confidence (as stated` and `ext://sys.stdout` to confirm there are no other dependants, for example `evals/live.py` transcripts or `capture.py`. Any test found asserting the old repr text is a scope finding to report, not something to edit silently.

## Risks, assumptions and gaps

- **Moving the console handler to standard error affects every entry point:** `demo.py`, `evals.live`, `capture.py` and the unwired `server/`. For these, log records stop appearing on standard output, but they still reach the terminal on standard error and the file is unchanged. Any test that captured log records from standard output would break. None was found in the files read; the grep above confirms.
- **Logging handler state.** Calling `configure_logging()` inside a capsys test binds the console handler to pytest's capture stream. That is why the test must reconfigure afterwards, or a later test may write to a closed stream.
- **The new block format, especially the `component <name>:` label, is a presentation choice.** It is constrained only to show the response's own values. Wording for "no grade stated" is likewise the harness's own phrasing, not contract text.
- **Assumption:** the review-folder responses stay in place. Tests read them by path as the request directs. They are evidence files, so moving them breaks the tests on purpose.
- **Out-of-scope rule.** Claim-eligibility wording (architecture §6 rule 5 and ADR-009) is not enforced by this change and remains open (ADR-020 consequences).

## Planning gaps

- GAP: console log routing — the request, `docs/decisions.md` (ADR-016, which covers levels only), `docs/architecture.md`, `RUNBOOK.md` and `logging_config.yaml` take no position on where console records go. The plan uses `ext://sys.stderr` because it is the only listed option that leaves both the log file's content and the `LOG_LEVEL` contract unchanged. Turning console logging off for the CLI alone is the alternative. The owner should confirm, and should decide whether this needs a new ADR (decisions.md is append-only and is not edited here).
- GAP: `weakest_input` shape — the schema (`contract/poc_evidence_contract.v0.3.0.schema.json`) declares `grade` and an optional `reference` but does not forbid other keys. Only `{"grade": …}` has been seen (L01, VC-POC-13). The plan renders `grade` and `reference` and ignores any undeclared key. If more keys appear, rendering them needs a definition.
- GAP: `Modelled` grade rendering — `docs/decisions.md` ADR-009 and `docs/architecture.md` §6 rules 4a and 5 require a `Modelled` grade to render with its passing backtest artifact and stratum. This plan does not build that, because the only `Modelled` question (VC-POC-14) is gated and never rendered today. The contract for this work item must state that it is out of scope.
- GAP: claim-language gating — `docs/architecture.md` §6 rule 5 and ADR-009 bind `claim_eligibility` together with the grade. The request limits scope to the confidence block. ADR-020 records that this obligation is still open on the pipeline.
- GAP: ADR-009 Consequences in `docs/decisions.md` still say answers show "axis confidence without the headline grade until the contract ships it". The contract has shipped it (v0.3.0 schema), so that line is now out of date. Updating an append-only log is not in scope, so the owner should decide whether a superseding note is needed.

## Size estimate

Four files: two production files with small, local edits (one rendering helper and one YAML stream line), the CLI test file, and one new release note. Scale is small, across two subsystems (the CLI entry point and logging configuration), plus tests and release notes. No `risk_floor` interface or data entry is touched: `run.py` is not in the `answer-object` interface, and `harness/usage.py` and `sql/**` are untouched.

```yaml
size_estimate:
  files: 4
  scale: small
  subsystems: [cli, logging, tests, releases]
  paths: ["run.py", "logging_config.yaml", "tests/unit/test_run_cli.py", "releases/_next/cli-confidence-grade-and-clean-stdout.md"]
  interfaces: []
  data: []
```
