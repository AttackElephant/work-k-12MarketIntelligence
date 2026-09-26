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

Do not commit, stage, reset or clean Git state. Leave changes in the working tree
for the harness's separate verification steps. Do not claim tests passed unless
they were actually run. Finish with a brief account of changed files, any checks
performed and any unresolved requirements.

Work item: 20260908T113155
Request: Make evals/live.py execute and grade genuinely isolated intent, narration, and end-to-
  end stages while preserving quota-aware checkpointing, resumability, and completed-trial
  semantics.
Component repository: /Users/shane/Projects/k-12Harness
Current contract: /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113155/acceptance/004-contract.yaml
Plan: /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113155/planning/001-plan.md

Declared editable files:
- evals/graders.py
- evals/live.py
- evals/replay.py
- evals/schema.py
- harness/answer_view.py
- harness/intent.py
- harness/narration.py
- tests/live/test_live_eval.py
- tests/unit/test_graders.py
- tests/unit/test_live_tier.py

Read-only context supplied to the executor transport:
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113155/acceptance/004-contract.yaml
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113155/planning/001-plan.md
- /Users/shane/Projects/k-12Harness/contract/PINNED.json
- /Users/shane/Projects/k-12Harness/contract/POC_CONTRACT_V0.3_SPEC.md
- /Users/shane/Projects/k-12Harness/contract/POC_EVIDENCE_CONTRACT.md
- /Users/shane/Projects/k-12Harness/contract/poc_evidence_error.v0.3.0.schema.json
- /Users/shane/Projects/k-12Harness/contract/poc_input_catalog.v0.3.0.schema.json
- /Users/shane/Projects/k-12Harness/contract/poc_question_registry.v0.3.0.json
- /Users/shane/Projects/k-12Harness/contract/poc_school_directory.v0.3.0.schema.json
- /Users/shane/Projects/k-12Harness/contract/question_schemas/v0.3.0/VC-POC-01.request.schema.json
- /Users/shane/Projects/k-12Harness/contract/question_schemas/v0.3.0/VC-POC-03.request.schema.json
- /Users/shane/Projects/k-12Harness/contract/question_schemas/v0.3.0/VC-POC-04.request.schema.json
- /Users/shane/Projects/k-12Harness/contract/question_schemas/v0.3.0/_index.json
- /Users/shane/Projects/k-12Harness/docs/decisions.md
- /Users/shane/Projects/k-12Harness/evals/cases/golden_cases.json
- /Users/shane/Projects/k-12Harness/prompts/input_catalog.snapshot.json
- /Users/shane/Projects/k-12Harness/prompts/rules.md

## Existing candidate correction

Continuation: C004
Starting candidate: 297428e92accc3f343b443c36ee01147eb7d82bb2363ceb48005d82192202e3e
Source: deterministic-failure

Start from the existing candidate; do not rebuild it or discard work that is right.
The plan and the current contract remain your obligations in full: every required output, observable behaviour and criterion, not only the failures listed below. The failures are what is known to be wrong; they are not the limit of the work. Check the candidate against the plan and the contract and complete whatever it still owes.
The declared edit scope and every recorded human decision remain authoritative.

Known failures:
- full-test-suite: exit code 1, expected 0; output contains '\\d+ failed' (log verification/V003/001-full-test-suite.log)
- graders-tests: exit code 1, expected 0; output contains '\\d+ failed' (log verification/V003/004-graders-tests.log)

Human guidance:

Follow the contract's grader design exactly: stage-scoped selection via INTENT_GRADERS and NARRATION_GRADERS and one dispatch, grade_case(case, record, stage=None), where stage=None reproduces GRADERS_BY_DIRECTION exactly and stage='intent' / stage='narration' select the stage sets. Remove the public dispatchers grade_intent and grade_case_narration from evals/graders.py, including their 'if intent is None' / 'if record is None' branches: they exist only to manufacture a mutation and are a workaround. Update evals/live.py to call grade_case(..., stage=...). The only new public grader is grade_intent_question_match; keep its wrong-question mutation in tests/unit/test_graders.py and fix its runner so failures() receives a list. Remove the ISOLATED_MUTATIONS entries for the removed dispatchers. Do not rename, alias, exempt or hide any grader, and do not change the exemption set in test_every_grader_has_at_least_one_mutation. Keep every PF-1..PF-4 change and the end-to-end legacy path intact.
