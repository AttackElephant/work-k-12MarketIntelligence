This all looks complete and correct.

## Summary

I implemented the corrections against the approved plan/contract, starting from candidate C002, within the declared edit scope.

**`evals/live.py`**
- Fixed the PF-1 defect: the narration oracle now *always* uses `build_client("stub", case.stub_case)`, ignoring `--emitter`; `--stage narration --emitter real` is refused at argument validation; narration eligibility (`select_cases`) is now gated on `case.stub_case is not None` (plus the fixed question/request needed to fetch it), the excluded count is printed, and an empty selection still exits non-zero via the existing `SystemExit("no cases selected")`.
- Fixed PF-2: on `CredentialsRejected`, `main()` now checks whether `report_path` already exists (a checkpoint from a completed trial) and, if so, prints the truthful message with the checkpoint path and the `--resume` command instead of always claiming "No report was written"; no extra write occurs, so an existing checkpoint is left untouched.
- Fixed contract-compatibility item: removed `run_end_to_end_trial`, which had wrongly routed `--stage end-to-end` through the new `ask(deps, question)` pipeline. `end-to-end` now falls through to the same legacy `ask(build_agent(), deps, phrasing)` path as the default stage, differing only via `main()`'s existing k=1 check.
- `run_narration_trial` signature simplified (dropped the now-meaningless `emitter` parameter).

**`tests/unit/test_live_tier.py`**
- Rewrote narration-selection tests for the `stub_case`-based eligibility rule, added tests for the excluded-count print, the non-zero exit on an empty narration selection, and the `--emitter real` refusal.
- Replaced the two tests that asserted `--stage end-to-end` used the new pipeline with tests proving it uses the same legacy path as default (including that `intent_model`/`narration_model` have no effect there).
- Added PF-2 tests (parametrized over all four stages) covering both the no-checkpoint and existing-checkpoint `CredentialsRejected` cases.
- Added PF-3 spy tests proving a narration trial never calls `interpret_intent`/builds an agent, an intent trial never calls `build_client`/`narrate`, and two narration runs differing only in scripted intent produce identical results (added a `_drop_generated_utc` helper to avoid flakiness from the stub emitter's wall-clock timestamp when comparing two independently-fetched envelopes).
- Added the PF-4 test asserting `tests/live/test_live_eval.py` collects zero tests under default options and every test under `-m live`.

**`tests/live/test_live_eval.py`**: updated the narration-stage case selection to use `stub_case` per PF-1.

**Not resolved — scope conflict (please review):** Human-guidance item 1 (restore public `grade_intent`, `grade_intent_question_match`, `grade_case_narration` names in `evals/graders.py`, backed by mutations in `tests/unit/test_graders.py`) could not be completed. `tests/unit/test_graders.py` is not in the declared editable-file set, and `test_every_grader_has_at_least_one_mutation` there scans `dir(graders)` for any public `grade_*` name and requires a matching entry in its hardcoded `MUTATIONS` table — renaming the isolated-stage graders to public names without editing that file would fail `graders-tests` and `full-test-suite`. I left the private-name/alias-at-import-site scheme in `evals/graders.py` unchanged and flag this as needing either an edit-scope amendment (adding `tests/unit/test_graders.py`) or a different resolution.

I was not able to execute `.venv/bin/python -m pytest …` myself (no shell/test-execution tool available in this session), so these changes are unverified by an actual test run — please run the deterministic gates in the contract before merging.
