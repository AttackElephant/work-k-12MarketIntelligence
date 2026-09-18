# Independent Requirements and Problem Framing

I read `root2:request.md` (identical wording to the referenced work-item request), the pinned contract set (`PINNED.json`, `POC_CONTRACT_V0.3_SPEC.md`, `POC_EVIDENCE_CONTRACT.md`, the v0.3.0 error/input-catalog/school-directory schemas, the question registry, the VC-POC-01/03/04 request schemas and `_index.json`), the decision log, the golden cases, the input-catalog snapshot, and the prompt rules. I requested `root0:work/20260908T113118/request.md`; the host did not return its contents, so this framing rests on the copy at `root2:request.md`. This stage carries **no repository/implementation evidence**; every claim about `harness/planner.py` below is a hypothesis for later inspection, not a finding.

## Authoritative requirements

The request names five obligations for `harness/planner.py`. Mapped to the Definition Pack:

1. **Deterministically derive required inputs.** Per-question `required`, `material`, `conditional_material`, `defaultable`, and `optional` are owned by the contract (`poc_question_registry.v0.3.0.json` `input_policy` and `question_schemas/v0.3.0/_index.json`). "Deterministically derive" implies the planner reads these from the contract rather than inventing or hardcoding them; ADR-001 forbids the harness computing anything, and ADR-008 pins the contract as the single interface. Missing **material** inputs must produce a clarification (prompt rules step 3; golden `should_clarify` cases with `expected_missing_inputs`); **defaultable** inputs must be left for the data layer to default and echo in `resolved_request` (contract "Input policy" and `resolved_request` requirement).

2. **Resolve schools safely.** Named schools go through the directory preflight (`--school-directory`, `poc_school_directory.v0.3.0.schema.json`); the planner submits the resolved `acara_sml_id`, never a name match (ADR-001, prompt rules step 1). Multiple matches -> ask the user; a directory record is identity only (not campus/School_No/catchment). An `acara_sml_id` absent from the pinned directory is `bad_request` (contract error boundary); a directory-valid record missing geography/coordinate is `unanswerable` — these are distinct outcomes the planner must not conflate.

3. **Normalize requested metrics.** Metrics must come from the correct allow-list — `profile_metrics` (VC-POC-01/04) vs `student_context_metrics` (VC-POC-11) per `x-poc-request-field-to-catalog-list` and `_index.json` — using the single-source `input_catalog.snapshot.json` (ADR-015). Prompt rules step 2 fixes semantics: "teaching staff" -> `teaching_fte`, headcount only when explicitly asked, non-teaching only when explicitly asked, and **omit `metrics` entirely when the user names none** (let the data layer default). The "smallest request" rule prohibits adding related-but-unrequested metrics.

4. **Construct exact evidence requests.** The built request must validate against the per-question request schema, which sets `additionalProperties: false`, closed-vocabulary `enum`s, `uniqueItems`, and cardinality (e.g. VC-POC-04 `minItems: 2`; VC-POC-09/10 registry rule "at least one explicit filter"; OD-07 `minProperties >= 1`). `request` must be exactly what the harness supplied and evidence filters must agree with `resolved_request` (contract "Successful envelope additions").

5. **Surface planner-generated bad requests as failures, not unsupported user questions.** The error boundary defines `bad_request` as harness-repairable (syntax/shape/value/directory identity). The request draws a line: when the *planner itself* emits a malformed request, that is a harness defect and must be surfaced as a failure — not silently rendered to the user as an `Unsupported` (i.e., "your question can't be answered"). This aligns with ADR-016's rule that an implementation failure must never be folded into a quality/refusal outcome, and ADR-009/ADR-014's insistence that unearned or misclassified outcomes be treated as invalid rather than rendered.

## Scope and compatibility boundaries

- **Single module.** The work item is scoped to `harness/planner.py`. The vendored contract is pinned by hash (ADR-008); the planner must not modify contract files, and the registry version must remain `0.3.0` (`PINNED.json`).
- **Contract v0.3.0 is the authority.** Adoption of v0.3.0 (O-1) obsoletes three interim shims: the hardcoded `settings.MATERIAL_INPUTS`, harness-side empty/partial detection, and axis-only confidence. The planner should derive materiality from the contract, not from the shim.
- **No computation, no name-matching, no invented values** (ADR-001). Every emitted request value must trace to the user's words, the directory, or the allow-lists.
- **Conditional materiality is a known trap.** O-8 records that `distance_km` is material *only when* `market_definition == distance`, and the interim gate treats it unconditionally (pinned by a strict `xfail`); O-6/G-4 warn against string-splitting registry prose. The clean source is the per-question schema's conditional rules (CR-5). The planner must not introduce a second hand-maintained condition table.
- **Refusals/gating are legitimate outcomes, not planner failures.** `excluded_question` (e.g., VC-POC-14 `not_yet_validated`, VC-POC-TRANSPORT-01, excluded topics) and `unanswerable`/empty results are correct user-facing outcomes and must remain distinct from the "planner-generated bad request" failure class.
- **Determinism boundary.** ADR-018/019 make the model a live, non-deterministic transport; "deterministically derive" suggests code-owned derivation of required inputs, but the planner may sit at the model/code seam. The exact division of labour is unknown at this stage.

## Initial hypotheses to test

All provisional — no implementation has been seen.

- **H1 — Contract-sourced materiality.** The planner derives required/material/defaultable inputs from the registry or `_index.json`, not from `settings.MATERIAL_INPUTS`. Risk: residual dependence on the interim shim (O-1).
- **H2 — Deterministic derivation.** Given identical inputs, the planner produces byte-identical structured requests (testable via `evals/replay.py`), rather than delegating derivation entirely to the model.
- **H3 — Safe school resolution.** The planner calls `search_schools` first, batches multiple named schools in one call, clarifies on multi-match, and never name-matches; it treats a directory-absent ID as `bad_request` and a directory-valid record missing geography as `unanswerable`.
- **H4 — Correct metric list selection and minimality.** The planner picks `profile_metrics` vs `student_context_metrics` per question, maps "teaching staff" -> `teaching_fte`, omits `metrics` when none are named, and never widens beyond the words used.
- **H5 — Schema-exact construction.** Emitted requests satisfy `additionalProperties: false`, enums, `uniqueItems`, and cardinality/mutual-exclusion rules (VC-POC-04 >=2 IDs; VC-POC-09/10 explicit-filter rule).
- **H6 — Failure-class separation.** A request the planner itself malforms surfaces as a distinct failure (e.g., internal/planner defect) rather than being converted to `Unsupported`; genuine `unknown_question`/`excluded_question`/`unanswerable`/empty remain user-facing outcomes.
- **H7 — Conditional `distance_km`.** The planner correctly treats `distance_km` as material only for `market_definition == distance` (VC-POC-05/06/07), and does not require it on SA2/LGA/postcode markets — the O-8 hazard.

## Required implementation and test evidence

A later current-state inspection should obtain:

- **`harness/planner.py` itself**, plus whatever it reads for materiality/defaults (registry loader, `contract.material_inputs_for()`, `_index.json`, `settings.MATERIAL_INPUTS`).
- **The request-construction path** and where the union output branches (answer / clarify / refuse / Unsupported) are chosen, and how `RepairCapExceeded`/`classify()` map emitter `error_kind`s to outcomes (ADR-003, ADR-014, ADR-016).
- **The failure taxonomy** distinguishing planner-generated `bad_request` from user-facing abstain outcomes; confirmation it is not folded into `Unsupported`.
- **Golden cases** (`evals/cases/golden_cases.json`) — the `expected_request`, `expected_missing_inputs`, `unordered_request_keys`, and refusal cases — and the graders that check chosen-question and request match.
- **Tests**: `tests/unit/test_agent_loop.py` (including the O-8 `xfail`), `test_rendering.py`, `test_graders.py`, `test_contract_pin.py`, and `evals/replay.py` for determinism.
- **Per-question request schemas** for all IDs (only 01/03/04 were provided here) to verify cardinality/conditional coverage — especially VC-POC-05/06/07 (`distance_km`), VC-POC-09/10 (filter minimums), VC-POC-12/13 (defaults), and VC-POC-14 (gated).

## Failure modes and acceptance hazards

- **Miscategorising absence.** Treating a well-typed, non-matching value (empty/partial result) or a directory-valid record missing geography (`unanswerable`) as `bad_request`, or vice versa — the contract's central classification rule. A planner that "repairs" an empty result is defective.
- **Silent defect absorption.** The core hazard the request targets: a planner-built malformed request being rendered as `Unsupported`, hiding a harness bug behind a user-facing refusal (ADR-016's "failure folded into a quality metric").
- **Metric over-inclusion / wrong list.** Adding related metrics (violates minimality) or selecting `profile_metrics` where `student_context_metrics` applies (G-5 ambiguity), or emitting `metrics` when none were named.
- **Unconditional `distance_km`.** Requiring a radius on non-distance markets (O-8) — a wasted clarification, pinned by an `xfail` that will fail the suite if "fixed" without updating the entry.
- **Brittle prose parsing.** Deriving conditional materiality by string-splitting registry prose (G-4) rather than reading schema conditions.
- **Non-determinism.** If required-input derivation is model-driven, identical questions may yield different requests, defeating "deterministically derive."
- **Cardinality/uniqueness breaches.** Emitting <2 IDs for VC-POC-04, duplicate array items, or filter sets lacking an explicit filter for VC-POC-09/10.
- **Refusal/gating drift.** Misclassifying a legitimate `excluded_question` (VC-POC-14 `not_yet_validated`, excluded topics) as a planner failure, or the reverse.
- **Shim residue.** Continued reliance on `settings.MATERIAL_INPUTS` or harness-side empty detection that O-1 says v0.3.0 adoption should delete.
- **Repair-cap interaction.** The repair loop (ADR-014) exhausting into an `Unsupported` for a request the planner never should have malformed — masking the defect rather than surfacing it.

I return analysis only. Whether the current `harness/planner.py` satisfies these requirements, and any decision to change code or accept the work item, remain host-owned.
