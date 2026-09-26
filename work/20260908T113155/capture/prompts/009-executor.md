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

Do not commit, stage, reset or clean Git state. Leave changes in the working tree
for the harness's separate verification steps. Do not claim tests passed unless
they were actually run. Finish with a brief account of changed files, any checks
performed and any unresolved requirements.

Work item: 20260908T113155
Request: Make evals/live.py execute and grade genuinely isolated intent, narration, and end-to-
  end stages while preserving quota-aware checkpointing, resumability, and completed-trial
  semantics.
Component repository: /Users/shane/Projects/k-12Harness
Current contract: /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113155/acceptance/003-contract.yaml
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
- tests/unit/test_live_tier.py

Read-only context supplied to the executor transport:
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113155/acceptance/003-contract.yaml
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

Continuation: C002
Starting candidate: c71f6a2c94b02b5bbc1e8d6993aadfa8e0528d7bd79ea47a2a2184cde8aa19ef
Source: contract-defect

Start from the existing candidate; do not rebuild it or discard work that is right.
The plan and the current contract remain your obligations in full: every required output, observable behaviour and criterion, not only the failures listed below. The failures are what is known to be wrong; they are not the limit of the work. Check the candidate against the plan and the contract and complete whatever it still owes.
The declared edit scope and every recorded human decision remain authoritative.

The contract was revised after this failure: full-test-suite were judged to be the contract's fault. The current contract governs; the other failures below still stand.

Known failures:
- full-test-suite: output contains 'failed' (log verification/V002/001-full-test-suite.log)

Human guidance:

The previous attempts ran blind and then corrected only failing tests; the plan's Planning review 1 resolutions (PF-1..PF-4) and the contract are your obligations in full. (1) Restore the public names grade_intent, grade_intent_question_match and grade_case_narration: renaming them to _-prefixed names to escape test_every_grader_has_at_least_one_mutation breaks ADR-013. Give each new grader a load-bearing mutation in tests/unit/test_graders.py MUTATIONS instead. (2) PF-1: the narration oracle is always build_client('stub', case.stub_case) with the case's recorded request, never the --emitter value; refuse --stage narration --emitter real at argument validation; narration eligibility is having a stub_case, print the excluded count, and exit non-zero when no case is eligible. (3) PF-2: after CredentialsRejected, say truthfully whether a checkpoint exists and, if so, print its path and the --resume command, instead of always 'No report was written'; leave an existing running checkpoint byte-for-byte unchanged; test both cases for every stage. (4) PF-3: spy tests proving a narration trial never calls interpret_intent or the evidence tool, an intent trial never fetches or calls narrate, and two narration runs differing only in scripted intent produce identical AnswerView, output and grades. (5) PF-4: a unit test that collecting tests/live/test_live_eval.py with default options selects zero tests and with -m live selects every test in the file. (6) Contract compatibility: --stage end-to-end and --stage omitted keep ask(build_agent(), deps, phrasing); do not switch end-to-end to the new pipeline.
