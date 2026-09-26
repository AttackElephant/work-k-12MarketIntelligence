This is consistent. The changes are complete and self-contained.

## Summary

Addressed the recontract's required fix (restore public graders + load-bearing mutations) while preserving the rest of the prior candidate's work.

**Changed files:**

1. **`evals/graders.py`**
   - Renamed `_intent_question_match` → `grade_intent_question_match` (public, no underscore-hiding trick, doc updated).
   - Renamed `_grade_intent` → `grade_intent`; it is no longer a bare wrapper — it now also owns a genuine guard (`intent is None` → its own `GradeResult("intent", False, ...)`), giving it load-bearing behavior distinct from the leaf grader it dispatches to.
   - Renamed `_grade_case_narration` → `grade_case_narration`; likewise gained its own `record is None` guard returning `GradeResult("case_narration", False, ...)`.
   - `grade_case`'s narration dispatch now calls the public `grade_case_narration`.

2. **`evals/live.py`**
   - Removed the aliasing import `from evals.graders import _grade_intent as grade_intent`; now imports the public `grade_intent` directly alongside `failures`, `grade_case`.

3. **`tests/unit/test_graders.py`**
   - Imports the three restored public names.
   - Added `ISOLATED_MUTATIONS` (a second, parallel mutation table for the non-`RunRecord`-shaped isolated-stage graders) plus `test_isolated_stage_mutation_is_caught_by_its_grader`, giving each of `grade_intent_question_match`, `grade_intent`, and `grade_case_narration` a real adversarial mutation that must be genuinely caught.
   - Updated `test_every_grader_has_at_least_one_mutation` to union coverage from both `MUTATIONS` and `ISOLATED_MUTATIONS`, without narrowing or altering its `dir(graders)` scan — it still scans every public `grade_*` symbol.

**Checks performed:** Read-through verification only (no shell access in this environment) — confirmed no remaining references to the old private/aliased names anywhere in the repo (`evals/`, `tests/`), confirmed `grade_case`, `grade_intent`, `grade_case_narration` call sites are consistent, and confirmed the new mutation table's expected grader labels (`intent_question_match`, `intent`, `case_narration`) match exactly what the corresponding functions return on the constructed failure inputs. I was not able to actually execute `pytest`, so the contract's deterministic gates (`tests/unit/test_graders.py`, `tests/unit/test_live_tier.py`, full suite) should be run by the harness's verification step to confirm.

**Unresolved:** None identified within the declared scope; all other files from the prior candidate (`evals/live.py` stage wiring, `evals/schema.py`, `evals/replay.py`, `harness/*`, `tests/live/test_live_eval.py`, `tests/unit/test_live_tier.py`) were left untouched since the known failure was scoped specifically to the graders/test_graders naming issue.
