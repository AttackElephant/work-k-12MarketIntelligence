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

Work item: 20260908T113118
Request: Make harness/planner.py deterministically derive required inputs, resolve schools
  safely, normalize requested metrics, construct exact evidence requests, and surface planner-
  generated bad requests as failures rather than unsupported user questions.
Component repository: /Users/shane/Projects/k-12Harness
Current contract: /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113118/acceptance/003-contract.yaml
Plan: /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113118/planning/002-plan.md

Declared editable files:
- harness/planner.py
- run.py
- tests/unit/test_intent_boundary.py
- tests/unit/test_planner.py
- tests/unit/test_run_cli.py

Read-only context supplied to Aider:
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113118/acceptance/003-contract.yaml
- /Users/shane/Projects/work/k-12MarketIntelligence/work/20260908T113118/planning/002-plan.md
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
