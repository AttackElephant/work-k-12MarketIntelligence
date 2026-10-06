<!-- Derived view of 002-contract.yaml. The YAML is authoritative. Regenerate; never edit. sha256=0350efd74f696ae5bd5056867bdc3ba9a19f7b289c185c20c11cf2efd9ef42f1 -->

# Acceptance contract — work item 20261005T143536

contract_version: 0.1.0  ·  artifact_version: 2

## Observable behaviour
- **l01-stated-grade** — Rendering the saved response docs/reviews/2026-10-poc-review/_work/envelopes/L01.json through run.render_text gives a confidence block with the line "grade: Observed" (from confidence.grade.value) and the line "weakest input: Observed" (from confidence.grade.weakest_input.grade). The block still has "forecast: not_applicable", "linkage: high", "source: high" and "temporal: medium". No line of the block contains "{" or "adjustments".
- **compound-component-grades** — Rendering docs/reviews/2026-10-poc-review/_work/envelopes/direct_VC-POC-13_579499.json gives a confidence block with "component sa2_age_projection: Indicative" and "component sa2_projection: Indicative", each read from confidence.grade.component_grades.<name>.value and listed in name order, as well as the headline "grade: Indicative" and "weakest input: Indicative" (architecture section 6, rule 5). No line of the block contains "{".
- **no-grade-stated** — When confidence has no grade, or the grade is not a mapping, or the grade has no string value, the block shows the axis ratings and one line "grade: not stated by the data layer". It shows no grade word (Observed, Derived, Modelled, Indicative, Unsupported), no weakest-input line and no component lines (ADR-009).
- **text-stdout-outcome-only** — run.py in its default text mode writes only the rendered outcome to standard output. No line of standard output is a log record (a JSON object with a "level" key); console log records go to standard error instead (DECIDED (C1)). The log file logs/harness.log, resolved under settings.PROJECT_ROOT, still receives the records.
- **json-stdout-one-document** — run.py --json writes exactly one JSON document to standard output, and that document is output.model_dump_json(indent=2), unchanged. json.loads of the whole of standard output succeeds.
- **heading-and-axes-preserved** — The heading "Confidence (as stated by the data layer):" and the "  - " bullet format stay word for word. Every confidence key other than grade whose value is a string, number or boolean renders as "<axis>: <rating>" in sorted order, as it does today.

## Required outputs
- `run.py` — The confidence block in render_answer is rebuilt through a private helper (for example _confidence_lines). The render_answer docstring and the ADR-009 sentence in the module docstring are updated to say the grade is shown when stated and never supplied.
- `logging_config.yaml` — Per DECIDED (C1), handlers.console.stream changes from ext://sys.stdout to ext://sys.stderr and the header comment says why. Levels, the formatter, file_main, the logger entries and the transport pins do not change.
- `tests/unit/test_run_cli.py` — test_confidence_renders_as_the_axes_the_data_layer_stated is replaced with a test of the ADR-009 rule for a stated grade. New tests cover L01, VC-POC-13, the parametrised cases with no grade stated, text-mode standard output with no log record (with a log-file check under tmp_path), and --json as one document. The module docstring changes from "with no overall grade added".
- `releases/_next/cli-confidence-grade-and-clean-stdout.md` — Release note copied from releases/_template.md and required by the release-notes hook and PR checks. Type minor, because run.py and logging_config.yaml are minor paths. It covers the summary, local deployment (nothing to do), manual steps (nothing to do), a verification block running .venv/bin/python -m pytest tests/unit/test_run_cli.py tests/unit/test_logging_config.py -q, and rollback.

## Deterministic criteria
- **l01-block-shows-stated-grade** in `k-12Harness`
  - command: `.venv/bin/python -c 'import json, run; from harness.outputs import AnswerDraft, RenderedAnswer; env = json.load(open("docs/reviews/2026-10-poc-review/_work/envelopes/L01.json")); out = RenderedAnswer.from_envelope(AnswerDraft(question_id=env["question_id"], answer="probe"), env); lines = [l.strip().lstrip("- ").strip() for l in run.render_text(out).split("Confidence (as stated by the data layer):")[1].split("\n\n")[0].strip().splitlines()]; print("\n".join(lines)); assert "grade: Observed" in lines, "stated grade line missing"; assert "weakest input: Observed" in lines, "weakest input line missing"; assert all(a in lines for a in ("forecast: not_applicable", "linkage: high", "source: high", "temporal: medium")), "an axis rating is missing"; assert not any("{" in l or "adjustments" in l for l in lines), "a printed dictionary is in the block"; print("L01-BLOCK-OK")'`
  - expected: Exits 0 and prints L01-BLOCK-OK. On the base commit it fails: render_answer prints confidence.grade as a Python dictionary ("grade: {'adjustments': [], ...}"), so there is no "grade: Observed" line and a block line contains "{". It passes once the helper renders grade.value, weakest_input.grade and the four ratings, and no repr.
  - machine-checked: yes
- **vc-poc-13-block-shows-component-grades** in `k-12Harness`
  - command: `.venv/bin/python -c 'import json, run; from harness.outputs import AnswerDraft, RenderedAnswer; env = json.load(open("docs/reviews/2026-10-poc-review/_work/envelopes/direct_VC-POC-13_579499.json")); out = RenderedAnswer.from_envelope(AnswerDraft(question_id=env["question_id"], answer="probe"), env); lines = [l.strip().lstrip("- ").strip() for l in run.render_text(out).split("Confidence (as stated by the data layer):")[1].split("\n\n")[0].strip().splitlines()]; print("\n".join(lines)); assert "component sa2_age_projection: Indicative" in lines, "sa2_age_projection grade missing"; assert "component sa2_projection: Indicative" in lines, "sa2_projection grade missing"; assert "grade: Indicative" in lines and "weakest input: Indicative" in lines, "headline grade missing"; assert not any("{" in l for l in lines), "a printed dictionary is in the block"; print("VC-POC-13-BLOCK-OK")'`
  - expected: Exits 0 and prints VC-POC-13-BLOCK-OK. On the base commit it fails because component_grades appears only inside the printed grade dictionary, with no line per component. It passes once each entry of confidence.grade.component_grades renders as "component <name>: <value>".
  - machine-checked: yes
- **no-stated-grade-shows-ratings-only** in `k-12Harness`
  - command: `.venv/bin/python -c 'import copy, json, run; from harness.outputs import AnswerDraft, RenderedAnswer; base = json.load(open("docs/reviews/2026-10-poc-review/_work/envelopes/L01.json")); a = copy.deepcopy(base); del a["confidence"]["grade"]; b = copy.deepcopy(base); del b["confidence"]["grade"]["value"]; blk = lambda env: [l.strip().lstrip("- ").strip() for l in run.render_text(RenderedAnswer.from_envelope(AnswerDraft(question_id=env["question_id"], answer="probe"), env)).split("Confidence (as stated by the data layer):")[1].split("\n\n")[0].strip().splitlines()]; blocks = [blk(a), blk(b)]; print(blocks); assert all(all(x in ls for x in ("forecast: not_applicable", "linkage: high", "source: high", "temporal: medium")) for ls in blocks), "an axis rating is missing"; assert all(any(l.startswith("grade:") and "not stated" in l for l in ls) for ls in blocks), "no-grade wording missing"; assert not any(w in l for ls in blocks for l in ls for w in ("Observed", "Derived", "Modelled", "Indicative", "Unsupported")), "a grade word was shown"; print("NO-GRADE-OK")'`
  - expected: Exits 0 and prints NO-GRADE-OK. On the base commit it fails. With grade removed, no line says a grade was not stated. With grade.value removed, the printed dictionary still carries Observed (current_ceiling, maximum_grade, weakest_input). It passes once a missing grade or value renders the ratings plus "grade: not stated by the data layer" and no grade word.
  - machine-checked: yes
- **text-mode-stdout-carries-no-log-record** in `k-12Harness`
  - command: `.venv/bin/python -c 'import asyncio, io, json, logging, pathlib, sys, tempfile; import run as cli; from config import settings; from harness.outputs import Unsupported; tmp = pathlib.Path(tempfile.mkdtemp()); settings.PROJECT_ROOT = tmp; settings.LOG_LEVEL = "DEBUG"; cli.require_credentials = lambda: None; cli.build_client = lambda use_stub: object(); cli.converse = lambda *a, **k: (logging.getLogger("harness.agent").warning("probe-log-record"), asyncio.sleep(0, result=Unsupported(reason_kind="not_in_registry", reason="probe refusal")))[1]; buf = io.StringIO(); real = sys.stdout; sys.stdout = buf; code = cli.main(["--no-interactive", "probe question"]); sys.stdout = real; out = buf.getvalue(); print(out); records = [l for l in out.splitlines() if l.strip().startswith("{") and "\"level\"" in l]; log_text = (tmp / "logs" / "harness.log").read_text(encoding="utf-8"); assert code == cli.EXIT_UNSUPPORTED, code; assert "probe refusal" in out, "outcome not printed"; assert not records, "log records on stdout: %r" % records; assert "probe-log-record" in log_text, "log file did not receive the record"; print("TEXT-STDOUT-OK")'`
  - expected: Exits 0 and prints TEXT-STDOUT-OK. The real configure_logging runs inside cli.main, and a probe record is logged on harness.agent. On the base commit it fails because the console handler is bound to sys.stdout, so the JSON record with "level" lands on standard output. It passes once console records go to standard error (DECIDED (C1): handlers.console.stream is ext://sys.stderr), the refusal is still printed and the record still reaches <PROJECT_ROOT>/logs/harness.log.
  - machine-checked: yes
- **json-mode-stdout-is-one-document** in `k-12Harness`
  - command: `.venv/bin/python -c 'import asyncio, io, json, logging, pathlib, sys, tempfile; import run as cli; from config import settings; from harness.outputs import Unsupported; tmp = pathlib.Path(tempfile.mkdtemp()); settings.PROJECT_ROOT = tmp; settings.LOG_LEVEL = "DEBUG"; cli.require_credentials = lambda: None; cli.build_client = lambda use_stub: object(); refusal = Unsupported(reason_kind="not_in_registry", reason="probe refusal"); cli.converse = lambda *a, **k: (logging.getLogger("harness.agent").warning("probe-log-record"), asyncio.sleep(0, result=refusal))[1]; buf = io.StringIO(); real = sys.stdout; sys.stdout = buf; code = cli.main(["--json", "--no-interactive", "probe question"]); sys.stdout = real; out = buf.getvalue(); doc = json.loads(out); log_text = (tmp / "logs" / "harness.log").read_text(encoding="utf-8"); assert code == cli.EXIT_UNSUPPORTED, code; assert doc == refusal.model_dump(mode="json"), doc; assert out == refusal.model_dump_json(indent=2) + "\n", "stdout is not exactly the document"; assert "probe-log-record" in log_text, "log file did not receive the record"; print("JSON-STDOUT-OK")'`
  - expected: Exits 0 and prints JSON-STDOUT-OK. On the base commit it fails: the probe log record is written to standard output before the document, so json.loads raises "Extra data". It passes once standard output in --json mode is exactly output.model_dump_json(indent=2) followed by a newline, with the document unchanged and the log file still receiving the record.
  - machine-checked: yes
- **release-note-present** in `k-12Harness`
  - command: `grep -Ei 'type.*minor' releases/_next/cli-confidence-grade-and-clean-stdout.md`
  - expected: Exits 0, printing the Type line. Fails on the base commit because the release note does not exist yet. Passes once the note, copied from releases/_template.md, declares Type minor.
  - machine-checked: yes
- **full-test-suite** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/ -q`
  - expected: The repository verify command exits zero with no failures, errors or unexpected passes. It includes the rewritten and new tests in tests/unit/test_run_cli.py, tests/unit/test_demo.py (heading, limitations, provenance, no renderer of its own) and tests/unit/test_logging_config.py.
  - machine-checked: yes
- **focused-cli-demo-logging-tests** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_run_cli.py tests/unit/test_demo.py tests/unit/test_logging_config.py -q`
  - expected: Exits zero before and after. The logging tests pass unchanged (harness.* loggers follow LOG_LEVEL; anthropic, httpx and httpcore stay at WARNING), and the demo tests still find the heading "Confidence (as stated by the data layer):".
  - machine-checked: yes
- **protected-files-unchanged** in `k-12Harness`
  - command: `git diff --exit-code b3e722ceef0e75c372db0c9c629a0e7ff8647579 -- tests/unit/test_logging_config.py harness/logging_config.py config/settings.py harness/outputs.py harness/rendering.py demo.py contract/ docs/architecture.md`
  - expected: Exits 0 with no diff. The logging tests, the logging loader, the LOG_LEVEL default, the output model, the rendering gate, the demo, the pinned contract and the architecture document are identical to the base commit.
  - machine-checked: yes

## Must not change
- The --json document: run.main prints output.model_dump_json(indent=2) for every branch, with the same content as before. Only interleaved log lines disappear from standard output.
- tests/unit/test_logging_config.py, unchanged and passing: harness.* loggers inherit the single LOG_LEVEL setting, and anthropic, httpx and httpcore stay pinned at WARNING (ADR-016 finding 3).
- The LOG_LEVEL default (DEBUG) in config/settings.py, the root-level injection in harness/logging_config.py, and the formatter, levels, file_main handler and logger entries in logging_config.yaml. The content of logs/harness.log is unchanged.
- Every rendered block other than confidence: the prose, data status, deterministic signals, limitations (verbatim), provenance, and the footer "Question <id> · contract <version>". Also render_clarification, render_unsupported and render_budget.
- The heading "Confidence (as stated by the data layer):" and the bullet format of the block.
- converse, main control flow, the exit codes (EXIT_BY_BRANCH), argument parsing, and operational errors written to standard error.
- RenderedAnswer.confidence still carries the envelope confidence whole (harness/outputs.py untouched). The change is in display only.
- The vendored contract/ files and contract/PINNED.json (ADR-008).
- docs/decisions.md, which is append-only and not edited by this work item (DECIDED (C2)). docs/architecture.md is not edited either.
- The saved review responses under docs/reviews/2026-10-poc-review/_work/envelopes/, which tests read in place and never modify.

## Failure behaviour
- confidence has no grade key, or grade is not a mapping. → The block shows the axis ratings and one line "grade: not stated by the data layer". No grade word appears, and the harness supplies no grade of its own, whether a placeholder or one worked out from the axes (ADR-009).
- confidence.grade is a mapping without a non-empty string value. → The same as a missing grade: ratings plus "grade: not stated by the data layer". weakest_input and component_grades are not shown, even when present.
- A grade value is stated, but weakest_input is missing or has no grade. → The line reads "weakest input: not stated by the data layer". No grade word is substituted.
- A component_grades entry has no value. → The line reads "component <name>: not stated by the data layer".
- A confidence key other than grade has a dict or list value, which the schema forbids. → It is never printed as a Python or JSON repr. The executor either renders "<axis>: (not shown — unexpected structure)" or skips it with a code comment. No block line contains "{".
- weakest_input carries keys other than grade and reference. → Only grade is rendered, plus " (<reference>)" when reference is a string. Undeclared keys are ignored and never printed as a repr.
- The command line fails operationally (HarnessError, EmitterFailure, ClaudeAccountCliError). → Unchanged: the message goes to standard error and the exit code is 1.

## Compatibility
- ADR-009: confidence renders as the data layer states it. After CR-4 that means the envelope grade and weakest input, verbatim. Every grade word in the block is copied from the response, and the harness never assigns a grade.
- Architecture section 6 rule 5: grade.value and weakest_input are the headline, and a compound answer shows each component with its own grade.
- ADR-016 finding 3: LOG_LEVEL is the single verbosity setting for harness.* loggers, and the transport libraries stay at WARNING.
- ADR-017 decision 3: the demo renders through run.render_text and defines no renderer of its own. The demo confidence block changes with this one, and its heading stays word for word.
- ADR-008: the contract pin is untouched. The grade shape is read as contract/poc_evidence_contract.v0.3.0.schema.json declares it (confidence.grade, weakest_input, component_grades).
- Per DECIDED (C1), every entry point that calls configure_logging (demo.py, evals.live, capture.py, the unwired server/) sends console log records to standard error instead of standard output. Records still reach the terminal and the log file is unchanged.
- The release-notes hook and PR checks require a release note under releases/_next/ for a change to minor paths. VERSION is not touched.

## Compatibility baselines
_none_

## Contract requirements
- **scope-is-the-file-list** — The permitted change set is exactly required_outputs plus permitted_files. A repository test found to depend on the old printed-dictionary text, or on log records appearing on standard output, is a scope finding to report. It is not something to edit silently or implement around.
- **decision-console-routing** — DECIDED (C1): console log records go to standard error for every entry point. handlers.console.stream in logging_config.yaml changes from ext://sys.stdout to ext://sys.stderr; the levels, the log file and the LOG_LEVEL default are unchanged. Options (b) (switch console logging off for the command line only, inside run.py) and (c) (change the LOG_LEVEL default) are not taken. Consequently logging_config.yaml is a required output, the compatibility entry on other entry points applies to demo.py, evals.live, capture.py and server/, observable_behaviour text-stdout-outcome-only names standard error as where console records go, and the release note summary says console log records now go to standard error.
- **decision-routing-adr** — DECIDED (C2): no ADR is written in this work item and docs/decisions.md is not edited; it stays in must_not_change and out of permitted_files. The console-routing choice is written back to the decisions log after acceptance, outside this work item.
- **json-mode-reading** — Conflict stated, not resolved silently. The request says --json standard output is "exactly one JSON document, the same as before this change". Before this change, log records were interleaved on standard output, so standard output was not one document. This contract reads "the same as before" as the content of the document (output.model_dump_json(indent=2)), which must not change. Removing the interleaved log lines is the change.
- **assumption-block-labels** — ASSUMPTION: the block labels are the ones the plan chose, "grade: <value>", "weakest input: <grade>[ (<reference>)]", "component <name>: <value>" and "grade: not stated by the data layer". They are the harness wording, not contract text, and the deterministic criteria check them literally.
- **assumption-not-rendered-grade-fields** — ASSUMPTION: adjustments, backtest, current_ceiling, maximum_grade, rule_id and component rule_id are not rendered. The request asks only for value, weakest input and component grades.
- **gap-weakest-input-shape** — GAP: only {"grade": ...} has been seen for weakest_input (L01, VC-POC-13). The schema declares grade and an optional reference and does not forbid other keys. Undeclared keys are ignored here, and rendering them would need a definition.
- **gap-adr-009-consequences** — GAP: the ADR-009 Consequences in docs/decisions.md still say answers show axis confidence "without the headline grade until the contract ships it". The v0.3.0 contract has shipped the grade, so the line is out of date. A superseding note is the owner's to make, and this work item does not make it.
- **assumption-evidence-files-in-place** — ASSUMPTION: the saved responses stay at docs/reviews/2026-10-poc-review/_work/envelopes/. Tests read them by path, as the request directs, so moving them breaks the tests on purpose.
- **preflight-grep** — Before editing, the executor does a read-only search for render_text, "Confidence (as stated" and ext://sys.stdout to confirm there are no other dependants, for example evals/live.py transcripts or capture.py. A dependant that would break is reported as a scope finding.

## Semantic criteria
- **grade-words-only-from-response** — Every grade word in the rendered block is copied from the response: confidence.grade.value, weakest_input.grade, or component_grades.<name>.value. No code path derives, defaults or infers a grade from the axes or from anything else.
- **rewritten-test-still-forbids-unstated-grade** — The test that replaces test_confidence_renders_as_the_axes_the_data_layer_stated limits every block label to the non-grade axes, plus "grade" and "weakest input", plus "component <name>" for each component the response states. It asserts that the grade and weakest-input lines equal the response values, so a renderer that added a grade the response did not state would fail it.
- **tests-use-saved-responses** — The L01 and VC-POC-13 tests build RenderedAnswer.from_envelope from the saved files under docs/reviews/2026-10-poc-review/_work/envelopes/, read in place. The tests for a missing grade use a deep copy of L01, not hand-written envelopes.
- **stdout-test-isolation** — The text-mode and --json tests monkeypatch settings.PROJECT_ROOT to tmp_path, so no test writes to the repository logs/harness.log. They patch require_credentials, build_client and converse and use the real configure_logging. Afterwards they call configure_logging() again, so that later tests do not write to a closed capture stream.
- **release-note-and-docstrings** — The release note states that the confidence block shows the stated grade, weakest input and component grades, and that console log records now go to standard error. It names no manual or deployment step. The render_answer docstring, the ADR-009 sentence in the run.py module docstring, and the logging_config.yaml header comment describe the new behaviour accurately and say nothing about "no overall grade".
- **change-is-local** — The run.py diff is limited to the confidence-block helper, its call site in render_answer, and docstrings. The logging_config.yaml diff is limited to the console stream line and its comment.

## Out of scope
- Rendering a Modelled grade with its backtest block, artifact reference and stratum (ADR-009, architecture section 6 rules 4a and 5). The only Modelled question, VC-POC-14, is gated and never rendered today.
- Gating claim language on claim_eligibility (trend wording, the "directional, not decision-grade" framing). It remains open on the pipeline under ADR-020.
- Any change to what is answered, to the --json document, to the log file content, to the LOG_LEVEL default or to the transport logger pins.
- Rendering adjustments, backtest, current_ceiling, maximum_grade or rule_id, or any undeclared weakest_input keys.
- Display rounding of values (ADR-025) and any change to the prose, data status, signals, limitations, provenance or footer blocks.
- Editing docs/decisions.md (including a superseding note for the ADR-009 Consequences, or an ADR for the console-routing choice, which is written back after acceptance under DECIDED (C2)) or revising docs/architecture.md.
- Changes to harness/outputs.py, harness/rendering.py, evals/graders.py, demo.py, server/ or the stub fixtures.
- Bumping VERSION or cutting a release.

## Base-commit dry run

Every deterministic criterion run once at `k-12Harness` @ b3e722ceef0e, before any change (2026-10-06T02:05:56Z). A flag means the result cannot depend on the change.

- **l01-block-shows-stated-grade** (behaviour) — ran, exit 1 on the base, expectation not met
- **vc-poc-13-block-shows-component-grades** (behaviour) — ran, exit 1 on the base, expectation not met
- **no-stated-grade-shows-ratings-only** (behaviour) — ran, exit 1 on the base, expectation not met
- **text-mode-stdout-carries-no-log-record** (behaviour) — ran, exit 1 on the base, expectation not met
- **json-mode-stdout-is-one-document** (behaviour) — ran, exit 1 on the base, expectation not met
- **release-note-present** (behaviour) — ran, exit 2 on the base, expectation not met
- **full-test-suite** (regression) — ran, exit 0 on the base, expectation met
- **focused-cli-demo-logging-tests** (regression) — ran, exit 0 on the base, expectation met
- **protected-files-unchanged** (regression) — ran, exit 0 on the base, expectation met

_no criterion flagged_
