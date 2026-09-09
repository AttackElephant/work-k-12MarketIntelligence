# Implementation Plan

## Bounded objective

Make Stage 2 planning deterministic and fail-closed by:

- deriving missing required/material inputs from the pinned question policy rather than trusting `StructuredIntent.missing_material_inputs`;
- resolving named schools only when each mention has one unambiguous directory match;
- converting requested metric labels to the question’s canonical catalog vocabulary and request field;
- building the smallest schema-shaped evidence request without invented defaults; and
- treating an emitter `bad_request` produced by this deterministic planner as a harness failure, not an `Unsupported` user outcome.

Keep the change within the planner pipeline. Do not alter the legacy agent-tool repair loop, evidence contract, narration, rendering, or data emitter.

## Definition Pack interpretation

- The supplied Definition Pack files match the SHA-256 revisions recorded in `manifest.yaml`.
- The v0.3 registry and `_index.json` define each question’s required, material, conditional-material, defaultable, and optional inputs.
- Defaultable values must remain omitted from the submitted request unless the user explicitly supplied them; the emitter owns their resolution and reports them in `resolved_request`.
- Named schools must be resolved through the pinned school-directory interface. Directory results establish identity only and cannot be treated as campus, catchment, or crosswalk evidence.
- Ambiguous directory results cannot be resolved by choosing the first record.
- Metrics, ratios, and similar static vocabularies are closed. Their canonical values come from the pinned input catalog, with the question-to-catalog mapping supplied by the input-catalog schema and registry.
- VC-POC-01 and VC-POC-04 use `profile_metrics`; VC-POC-03 uses the `ratios` request field; VC-POC-11 uses `student_context_metrics`.
- A `bad_request` means the harness supplied a mechanically invalid request. In a deterministic planner with no repair model, that is an implementation/intent-boundary failure and must not be represented as an unsupported user question.
- `unknown_question`, `excluded_question`, and `unanswerable` remain typed abstentions. `data_unavailable` and `internal` remain operational failures.

## Current implementation

- `harness/planner.py` asks for clarification only when the intent model has already populated `missing_material_inputs`; it does not independently compare the assembled inputs with the contract policy.
- Conditional materiality is not evaluated against the actual request. The planner can therefore miss a required radius or accept unnecessary missing-input claims.
- School names are queried sequentially, and the first returned record is accepted. Empty and ambiguous result sets do not stop evidence dispatch.
- Resolved IDs are accumulated directly in shared `Deps` state, making request construction vulnerable to stale or duplicate IDs if the same dependencies object is reused.
- IDs supplied in `filters` silently override resolved IDs through `setdefault`, even when the two sources conflict.
- `requested_metrics` is copied verbatim into a `metrics` field for every question. Natural-language labels are not normalized, catalog membership is not checked, and VC-POC-03 receives the wrong field (`metrics` instead of `ratios`).
- The scope cleanup for headcount and non-teaching metrics happens after request construction and can leave an invalid or over-broad metric list.
- An emitter `bad_request` currently becomes `Unsupported(reason_kind="out_of_scope")`, incorrectly presenting a planner defect as a user-facing refusal.
- Existing planner tests cover broad branch selection but explicitly encode unsafe first-match resolution and the obsolete `bad_request → Unsupported` behavior.

## Proposed changes

- `harness/planner.py`
  - Add a deterministic input-derivation step that operates on the question’s pinned `input_policy` and the values actually present or derivable from the intent.
  - Treat school selector fields as satisfiable from either explicit IDs or successful named-school resolution; derive all other missing material fields from the assembled filters rather than the model’s missing-input claim.
  - Evaluate `distance_km` as conditionally material only when `market_definition` is `distance`.
  - Continue omitting defaultable fields when the user did not supply them.
  - Return `ClarificationNeeded` for genuinely absent user-selectable inputs, with stable ordering, contract field names, catalog-backed allowed values where available, and no evidence call.
  - Resolve all named schools as one logical preflight operation, retaining mention order in the resulting ID list.
  - Require exactly one directory result per mention. Return clarification for no match or multiple matches rather than selecting a candidate.
  - Build resolved IDs in local planner state, deduplicate without reordering, and update `Deps` only after resolution succeeds.
  - Detect conflicts between explicit `acara_sml_id`/`acara_sml_ids` filters and directory-resolved identities; fail before dispatch rather than silently preferring one source.
  - Enforce the scalar selector for single-school questions and ordered unique arrays for multi-school questions, including VC-POC-04’s minimum of two schools.
  - Load/cache the input catalog through the existing client/dependencies path when requested metric normalization is needed, and record the preflight in `deps.tool_calls`.
  - Normalize metric labels deterministically by case-folding and separator normalization, preserving already-canonical values, and applying only explicitly authorised semantic aliases such as `teaching staff → teaching_fte`.
  - Select the vocabulary and request key from the pinned question mapping: `metrics` for profile/student-context questions and `ratios` for VC-POC-03.
  - Deduplicate normalized selectors while preserving user order, enforce the applicable closed vocabulary, and apply the existing headcount/non-teaching scope restriction before dispatch.
  - Construct a fresh request containing explicit intent filters plus only the planner-owned selector field appropriate to the chosen question. Do not inject presentation metadata, inferred defaults, or a generic `metrics` field.
  - Change the `bad_request` branch to raise `TerminalError` with question and emitter context after recording the failed invocation. Preserve the existing abstention and operational branches for all other error kinds.
  - Update module and function documentation to distinguish deterministic planner failures from the legacy model-driven repair loop.

- `tests/unit/test_planner.py`
  - Replace the first-match school-resolution expectation with deterministic unique, empty, ambiguous, duplicate, and conflicting-identity cases.
  - Add a recording directory client that can return different candidate sets per school mention and assert that evidence is never fetched after failed resolution.
  - Verify that multi-school mentions produce ordered unique `acara_sml_ids`, while single-school questions produce exactly one `acara_sml_id`.
  - Verify required/material inputs are derived even when `missing_material_inputs` is empty or incorrect.
  - Verify irrelevant model-reported missing fields are ignored, defaults are not requested, and `distance_km` is requested only for a distance market.
  - Cover canonical metric passthrough, separator/case normalization, the authorised teaching-staff alias, stable deduplication, question-specific catalog selection, and VC-POC-03 routing to `ratios`.
  - Assert exact dictionaries passed to `fetch_evidence` for VC-POC-01, VC-POC-03, and VC-POC-04, including absence of defaulted or unrelated fields.
  - Replace `test_bad_request_returns_unsupported` with an assertion that `TerminalError` is raised, the failed tool invocation is retained, and no `Unsupported` branch is produced.
  - Retain regression coverage for exclusions, gated questions, unknown questions, `unanswerable`, `data_unavailable`, and `internal`.

- `tests/unit/test_intent_boundary.py`
  - Add pipeline-level acceptance cases proving that planner-derived clarification bypasses answer-view/narration stages.
  - Add one end-to-end deterministic case for natural-language metric normalization and exact request construction.
  - Add a pipeline assertion that planner-generated `bad_request` propagates as a failure rather than becoming a user-facing `Unsupported`.

## Verification approach

Run the repository’s declared deterministic suite:

```bash
.venv/bin/python -m pytest tests/ -q
```

Targeted verification should additionally cover:

```bash
.venv/bin/python -m pytest tests/unit/test_planner.py tests/unit/test_intent_boundary.py -q
```

Relevant acceptance cases:

1. A VC-POC-05 intent lacking `market_definition` clarifies even when the model reports no missing inputs.
2. A VC-POC-05 SA2 request does not ask for `distance_km`; the same question with `market_definition="distance"` does.
3. A uniquely matched school produces one canonical ID; zero or multiple matches produce clarification and no evidence dispatch.
4. Two uniquely matched schools produce the correct ordered `acara_sml_ids`; duplicate resolution cannot satisfy VC-POC-04’s two-school constraint.
5. Explicit and directory-derived IDs must agree.
6. `["Total Enrolments", "teaching staff"]` becomes `["total_enrolments", "teaching_fte"]` for VC-POC-01.
7. A staffing-intensity selector becomes `ratios=[...]` for VC-POC-03, with no `metrics` property.
8. VC-POC-04 dispatch contains only its IDs and requested canonical metrics; year defaults remain omitted.
9. Unsupported, gated, unknown, and unanswerable questions preserve their existing typed outcomes.
10. Any emitter `bad_request` reached from the deterministic planner raises `TerminalError` and remains visible in the recorded invocation.

## Risks, assumptions and gaps

- The captured pack provides request schemas only for VC-POC-01, VC-POC-03, and VC-POC-04. It is insufficient for generic local JSON Schema validation of every registered question. This plan therefore derives policy from the captured registry/index and limits exact schema assertions to those three supplied schemas. Full pre-dispatch validation across all questions requires adding every request schema to the Definition Pack.
- The conditional-material registry values are descriptive strings rather than a structured predicate language. The only captured condition needed here is `distance_km` when `market_definition` equals `distance`. New condition forms should trigger a definition escalation rather than ad hoc parsing.
- The Definition Pack supplies canonical metric identifiers but no general natural-language synonym registry. Normalization must be limited to mechanical case/separator conversion and explicitly stated aliases from `prompts/rules.md`. Broader synonym handling requires a versioned authority source.
- `golden_cases.json` says an explicitly supplied ACARA ID `99999` requires no directory lookup and may return an empty success, while the v0.3 specification says an ID absent from the pinned directory is `bad_request`. Because the supplied directory fixture does not contain `99999`, direct-ID preflight validation cannot satisfy both authorities. This plan leaves explicit numeric IDs unchanged and applies safe resolution to named schools; the conflict should be resolved in the Definition Pack before enforcing directory membership for caller-supplied IDs.
- The school-directory contract does not define ranking or fuzzy-match confidence. Any result count other than exactly one must therefore clarify; the planner must not invent matching heuristics.
- Raising `TerminalError` for `bad_request` is intentionally stricter than the legacy tool loop, which has a model capable of repairing a request. Tests must keep these two execution paths distinct.

## Size estimate

- Scale: **medium**
- Files: 3
- Subsystems: deterministic planner, school-directory preflight handling, input-catalog normalization, pipeline error propagation, unit/integration tests
- Rough change: approximately 150–250 implementation lines plus 200–300 lines of focused deterministic tests.