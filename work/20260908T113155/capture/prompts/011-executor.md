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

Continuation: C003
Starting candidate: c8b1f305fd0dff0bf6b8ed930a92dbed52108aa027feb151aadddf2a4b3a3499
Source: contract-defect

Start from the existing candidate; do not rebuild it or discard work that is right.
The plan and the current contract remain your obligations in full: every required output, observable behaviour and criterion, not only the failures listed below. The failures are what is known to be wrong; they are not the limit of the work. Check the candidate against the plan and the contract and complete whatever it still owes.
The declared edit scope and every recorded human decision remain authoritative.

The contract was revised because its scope omitted files the work requires: k-12Harness:tests/unit/test_graders.py. The current contract permits them; edit them as the plan and the current contract require, and do not implement around the tests or registries they hold.

Known failures:
- scope omits k-12Harness:tests/unit/test_graders.py, which the work requires

Human guidance:

Restore the public graders grade_intent, grade_intent_question_match and grade_case_narration in evals/graders.py: public names, no aliasing, no private _grade_* names, no wrappers. Give each a load-bearing mutation in tests/unit/test_graders.py MUTATIONS so that test_every_grader_has_at_least_one_mutation passes because every public grader is genuinely covered, not because the scan stops seeing them. Every other plan and contract obligation remains yours in full.
