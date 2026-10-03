# Implementation Plan

## Bounded objective

This change touches `k-12Harness` only. When `run.py` prints a `RenderedAnswer` as text:

1. **Confidence block.** The block "Confidence (as stated by the data layer):" shows:
   - `confidence.grade.value`;
   - `confidence.grade.weakest_input` (its `grade`, plus its `reference` when present);
   - each entry of `confidence.grade.component_grades`, with that component's own value;
   - the four axis ratings, as the data layer stated them.

   No line of the block holds a printed Python or JSON dict, and nothing from `adjustments` appears. When there is no grade, or the grade has no `value`, the block shows the axes only and says that no grade was stated. The harness never supplies a grade (ADR-009).
2. **Clean standard output.** In default text mode, nothing on standard output is a log record (a JSON object with a `"level"` key). `logs/harness.log` still gets every record. `--json` output is exactly one JSON document, unchanged.

Out of scope:
- what is answered, and the `--json` payload;
- what the log file records;
- every other block (answer, data status, signals, limitations, provenance, the footer);
- `Modelled` and backtest display;
- `claim_eligibility` language gating;
- `demo.py`, `server/` and `evals/`.

## Definition Pack interpretation

- **Contract schema (authority 3), `confidence.grade`.** Required fields are `value`, `current_ceiling`, `maximum_grade`, `rule_id`, `weakest_input` and `backtest`. `weakest_input` requires `grade` and may carry `reference`. Its `additionalProperties` is not closed, so other keys are possible. `component_grades` is an object mapping a name to `{value, rule_id}`, with `minProperties: 1`. `adjustments` is an optional array. `confidence` has `additionalProperties: false`, so it holds exactly the four axes plus `grade`.
- **architecture.md §6 rule 5 (authority 2), post-CR-4 rule.** "`grade.value` + `weakest_input` render as the headline; compound answers render each component's own grade; a `Modelled` figure renders with the `backtest` block that earned it." The `claim_eligibility` part of the same rule is a language rule about the prose, not about the confidence block.
- **decisions.md ADR-009 (authority 1, wins on conflict).** Post-CR-4, the harness renders "the envelope's grade and weakest-input, verbatim" and never assigns a grade. ADR-016 finding 3 says `harness.*` loggers follow `LOG_LEVEL`, and the transport loggers are pinned at WARNING. ADR-017 point 3 says the demo renders only through `run.render_text`.
- **O-1** lists "axis-only confidence rendering (ADR-009 interim)" as a shim to delete once the contract carries a grade. Contract v0.3.0 carries one, so this change removes that shim.
- **Saved envelopes (authority 9) are the acceptance inputs.**
  - `L01.json`: `value`=Observed, `weakest_input`={grade: Observed}; axes forecast=not_applicable, linkage=high, source=high, temporal=medium; `adjustments`=[].
  - `direct_VC-POC-13_579499.json`: `component_grades` = {sa2_age_projection: Indicative, sa2_projection: Indicative}.
- **Logging.** No Definition Pack reference says where console records should go (see the gaps).

## Current implementation

- `run.py` `render_answer` builds `axes = [f"{axis}: {rating}" for axis, rating in sorted(output.confidence.items())]`. Because v0.3.0 `confidence` contains `grade`, this prints `grade: {'adjustments': [], 'backtest': {...}, ...}`. That is the dict the request complains about. The docstring still states the interim "no overall grade" rule.
- `RenderedAnswer.confidence` is `dict[str, Any]`. It is copied from the envelope in code (`from_envelope`), so the renderer receives an unvalidated grade shape and must be defensive.
- `tests/unit/test_run_cli.py::test_confidence_renders_as_the_axes_the_data_layer_stated` asserts `f"{axis}: {rating}"` for every key, including `grade`. It also asserts that the set of rendered keys equals `set(output.confidence)`. Its module docstring says "with no overall grade added".
- `logging_config.yaml`: the `console` handler is `StreamHandler` → `ext://sys.stdout`, at level DEBUG. Root and the `harness.retry`, `harness.agent`, `harness.usage` and `server` loggers all route to `console` and `file_main`. Other `harness.*` loggers, such as `harness.evidence`, propagate to root. `LOG_LEVEL` defaults to DEBUG, so a default `run.py` run puts JSON log lines on stdout around the answer.
- `run.main` calls `configure_logging()` first. It prints `render_text(output)` or `output.model_dump_json(indent=2)` to stdout, and prints errors to stderr.
- `demo.py` sets `LOG_LEVEL=WARNING` as a default and renders through `cli.render_text`. `tests/unit/test_demo.py` checks only the confidence heading.

## Proposed changes

1. **`run.py`**, rendering of the confidence block only.
   - Add a private helper, `_confidence_lines(confidence: dict) -> list[str]`. It must not be named `render_*`; that keeps the demo's no-second-renderer test meaningful. `render_answer` uses it in place of the current `axes` line. The heading stays `"Confidence (as stated by the data layer):"`.
   - Line order:
     - `grade: <value>`, only when `confidence["grade"]` is a dict whose `value` is a non-empty string.
     - `weakest input: <weakest_input.grade>`, with ` (reference: <reference>)` added when `reference` is a string.
     - One `component <name>: <value>` line per `component_grades` entry, sorted by name and only for entries whose `value` is a string.
     - The axis lines `<axis>: <rating>`, for every key other than `grade` whose value is a string, sorted (today's format).
   - When the grade or its `value` is missing or malformed, emit `grade: not stated by the data layer` and no other grade lines. Never fall back to `current_ceiling` or `maximum_grade`, and never derive a grade from the axes.
   - Do not render `adjustments`, `backtest`, `rule_id`, `current_ceiling` or `maximum_grade`. They stay intact in `--json`.
   - Never call `str()` or `json.dumps` on a non-string confidence value.
   - Update the `render_answer` docstring to the post-CR-4 rule in ADR-009.
   - Leave `main`, `converse`, the `--json` branch and all other renderers unchanged.
2. **`logging_config.yaml`**, console handler only. Change `stream: ext://sys.stdout` to `ext://sys.stderr`, and update the header and handler comments to say console records go to stderr so stdout carries only the outcome.
   - Not changed: levels, handler membership, the `file_main` handler, the WARNING pins, and `LOG_LEVEL`.
   - This is the recommended option. It depends on the first GAP below; if the owner chooses another option, this file entry changes.
3. **`tests/unit/test_run_cli.py`**:
   - Replace `test_confidence_renders_as_the_axes_the_data_layer_stated` with a test of the ADR-009 stated-grade rule:
     - Load `docs/reviews/2026-10-poc-review/_work/envelopes/L01.json` and build `RenderedAnswer.from_envelope(AnswerDraft(...), envelope)`.
     - Assert that the block contains `grade: Observed`, `weakest input: Observed`, `forecast: not_applicable`, `linkage: high`, `source: high` and `temporal: medium`.
     - Assert that no line in the block contains `{` or `adjustments`.
     - Assert that the set of rendered labels equals exactly the four axes plus `grade` and `weakest input`. This replaces the old "nothing added" check.
   - Add a VC-POC-13 test using `direct_VC-POC-13_579499.json`. It asserts `component sa2_age_projection: Indicative` and `component sa2_projection: Indicative`.
   - Add a no-grade test with two cases: `grade` removed, and a `grade` without `value`. Each asserts that the block shows the four axes and "grade: not stated by the data layer", and that no grade-vocabulary word (Observed, Derived, Modelled, Indicative, Unsupported) appears in the block.
   - Add `test_text_mode_stdout_holds_no_log_records`:
     - Monkeypatch `settings.PROJECT_ROOT` to `tmp_path` so the file handler writes to `tmp_path/logs/harness.log`.
     - Keep the real `configure_logging`, so the handler binds the stream captured by `capsys`.
     - Use a fake `converse` that logs through `harness.agent` and `harness.evidence` with a unique marker, then returns the L01 answer.
     - Assert that no stdout line parses as a JSON object with a `"level"` key.
     - Assert that the marker reached the tmp log file.
     - Restore logging afterwards with `configure_logging()`.
   - Add `test_json_mode_stdout_is_exactly_one_document`: the same setup with `--json`. Assert that `json.loads(capsys.readouterr().out)` succeeds and equals `output.model_dump(mode="json")`.
   - Update the module docstring point 1.
   - Leave all other tests untouched.
4. **`releases/_next/confidence-grade-and-clean-stdout.md`**. CI (`pr-checks.yml`) and `.claude/hooks/require-release-notes.sh` require exactly one note per PR.
   - Write it from `_template.md`, with `**Type:** minor` (a behaviour change in CLI output and where console logs go).
   - Include Local deployment ("Nothing to do."), Manual steps ("Nothing to do."), and a Verification ```bash``` block: `git checkout main && git pull`, then `.venv/bin/python -m pytest tests/unit/test_run_cli.py tests/unit/test_logging_config.py -q`.

Enforcement checks reviewed:
- **No change needed:**
  - `tests/unit/test_logging_config.py`: must pass unchanged.
  - `tests/unit/test_demo.py`: the heading is kept, and no `render_*` is added to `demo.py`.
  - `tests/unit/test_contract_pin.py`: no contract file is touched; the review envelopes are not pinned.
- **Not relevant:** this change adds no grader, mutation table, schema registry or `__all__` entry, so `tests/unit/test_graders.py` and `tests/unit/test_rendering.py` gain nothing.
- **Explicitly out of scope:** `VERSION`, `docs/decisions.md`, `docs/architecture.md`, `RUNBOOK.md` and `demo.py`.

## Verification approach

- Full deterministic suite: `.venv/bin/python -m pytest tests/ -q` (the manifest `verify` command).
- Focused tests: `.venv/bin/python -m pytest tests/unit/test_run_cli.py tests/unit/test_logging_config.py tests/unit/test_demo.py -q`.
- How each acceptance case is checked:
  - "What should happen" 1 and 2: the L01 and VC-POC-13 rendering tests.
  - "What should happen" 3: the text-mode stdout test, including the log-file assertion.
  - Refusals 1 and 2: the no-grade test and the `--json` single-document test.
  - Refusal 3: `test_logging_config.py` unchanged and green.
- Non-vacuity checks for the executor to report:
  - On the base commit, the new L01 test must fail because the dict line contains `{`.
  - On the base commit, the stdout test must fail with a JSON `"level"` line present.
  - Confirm that the stdout test emits at least one record at a level the default `LOG_LEVEL` enables.
- Read-only grep for tests that assert log output on stdout through `capsys` (for example `rg -n "capsys" tests/`). Any such test that depends on the console stream is a scope finding to report, not something to edit around.

## Risks, assumptions and gaps

- **All entry points are affected.** Redirecting the console handler in the YAML changes every entry point (`demo.py`, `evals/live.py`, `capture.py`, `server/`), not just `run.py`. The effect is also benign there (stdout gets cleaner), but it reaches beyond the stated CLI scope. A CLI-only alternative is to repoint the console handler in `run.main` after `configure_logging()`. That puts logging policy outside the config file, which `harness/logging_config.py` names as the single place for it.
- **Grade fields deliberately not rendered.** The block omits `rule_id`, `current_ceiling`, `maximum_grade` and `backtest`. The request, ADR-009 and §6.5 name only the value, the weakest input and the components. They remain in `--json`.
- **Stub-fixture assumption.** It is not verified whether the stub fixture's `confidence` (behind `answered()`) carries a grade. The new tests use the saved real envelopes and an explicit no-grade copy, so they do not depend on it.
- **Bad grade shape is not flagged.** A malformed grade is rendered as "not stated", not as an error. Validating the envelope shape is not this change's job.
- **Test file location.** `dictConfig` binds `ext://` streams when it runs, so the stdout test must call `configure_logging` inside the test while `capsys` is active. Monkeypatching `settings.PROJECT_ROOT` redirects only the handler file path; the config path was bound at import. That keeps tests from writing to the repository's `logs/harness.log`.

## Planning gaps

GAP: request.md "Gaps not yet settled" vs docs/decisions.md (ADR-016) vs docs/architecture.md vs logging_config.yaml: no Definition Pack reference says where console log records go. The plan recommends sending the console handler to stderr in `logging_config.yaml`, which affects every entry point. The owner must choose between this, CLI-only rerouting in `run.py`, and changing the default level. The last option is out of scope because it changes what the log file records.
GAP: docs/decisions.md (append-only ADR log) vs this change: whether moving console logging off stdout needs a new ADR is not settled. This plan does not edit decisions.md.
GAP: contract/poc_evidence_contract.v0.3.0.schema.json (`weakest_input` is not closed) vs request.md: only `{"grade": ...}` has been seen. The plan renders `grade` and the schema-declared `reference` and ignores any other key. Whether other keys must be shown verbatim is unresolved.
GAP: docs/architecture.md §6 rule 4a and rule 5, and docs/decisions.md ADR-009 (a Modelled grade renders only with its passing backtest) vs request.md (out of scope): the renderer gets no Modelled or backtest handling. A Modelled envelope reaching `render_text` would show "grade: Modelled" without its backtest block. The contract must record this exclusion.
GAP: docs/architecture.md §6 rule 5 (claim_eligibility gates language) and the ADR-020 consequences (claim eligibility not checked) vs request.md scope (confidence block only): claim-language gating stays out of scope and unaddressed.
GAP: docs/decisions.md ADR-009 ("grade and weakest-input, verbatim") vs contract schema (the grade also carries `rule_id`, `current_ceiling`, `maximum_grade`, `backtest`, `adjustments`): the plan reads "verbatim" as rendering the stated values without paraphrase, not every field of the grade. The owner must confirm that omitting the other fields from the text block is acceptable.

## Size estimate

This is a small, local change. It is one rendering helper in `run.py` and a one-line stream change, with comments, in `logging_config.yaml`. Most of the work is in `tests/unit/test_run_cli.py`: one test replaced and four added. The release note required by CI makes the fourth file. No contract, prompt, fixture or grader changes.

```yaml
size_estimate:
  files: 4
  scale: small
  subsystems: [cli, logging-config, tests, releases]
```
