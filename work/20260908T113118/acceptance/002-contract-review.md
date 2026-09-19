<!-- Derived view of 002-contract.yaml. The YAML is authoritative. Regenerate; never edit. sha256=081b344cd86fc83d57bd96e9d071d9b1df307bdccb478513c412bfa784c6214c -->

# Acceptance contract — work item 20260908T113118

contract_version: 0.1.0  ·  artifact_version: 1

## Observable behaviour
- **planner-derived-inputs** — The planner derives absent required, material, and applicable conditional-material inputs from the pinned question input_policy and the request values actually present or safely derivable; it does not trust StructuredIntent.missing_material_inputs as authoritative.
- **conditional-distance** — For VC-POC-05, VC-POC-06, and VC-POC-07, distance_km is missing only when market_definition is distance; non-distance market definitions do not require it.
- **omit-unsupplied-defaults** — Defaultable fields, including year ranges and default metric or ratio selections, remain absent from the submitted request unless explicitly supplied by the user.
- **stable-clarification** — Genuinely absent user-selectable inputs return ClarificationNeeded in pinned policy order, using contract field names and catalog-backed allowed values where available, without calling fetch_evidence.
- **safe-school-resolution** — Every named school must produce exactly one directory record before evidence dispatch; zero or multiple matches return clarification and do not select a candidate.
- **transactional-school-resolution** — Named schools are resolved as one logical preflight: IDs are accumulated locally in mention order, deduplicated without reordering, and committed to Deps only after all mentions resolve successfully.
- **selector-shape** — Single-school questions submit exactly one acara_sml_id; multi-school questions submit ordered unique acara_sml_ids; VC-POC-04 requires at least two distinct IDs.
- **identity-conflict** — Explicit acara_sml_id or acara_sml_ids values must agree with identities resolved from school mentions; disagreement fails before evidence dispatch.
- **metric-normalization** — Requested metric labels are case-folded and separator-normalized, canonical values pass through unchanged, and only explicitly authorised semantic aliases are applied, including teaching staff to teaching_fte.
- **metric-routing** — VC-POC-01 and VC-POC-04 use profile_metrics and submit metrics; VC-POC-03 uses ratios and submits ratios with no metrics property; VC-POC-11 uses student_context_metrics and submits metrics.
- **metric-scope** — Normalized selectors are validated against the applicable closed catalog vocabulary, deduplicated while preserving user order, and subject to the existing headcount and non-teaching restrictions before request construction.
- **exact-request** — The evidence request is freshly constructed from explicit intent filters and only the planner-owned selector appropriate to the question, without presentation metadata, inferred defaults, stale dependency state, or unrelated fields.
- **bad-request-failure** — An emitter bad_request reached through deterministic planning records the failed fetch_evidence invocation and raises TerminalError with question and emitter context; it never returns Unsupported.

## Required outputs
- `harness/planner.py` — Deterministic input derivation, transactional school resolution, metric normalization and routing, exact request construction, and fail-closed bad_request handling.
- `tests/unit/test_planner.py` — Focused unit coverage for input policy, school resolution, selector conflicts and cardinality, catalog normalization, exact requests, and all evidence-error branches.
- `tests/unit/test_intent_boundary.py` — Pipeline coverage proving planner clarification bypasses answer-view and narration, normalized requests reach evidence, and planner bad_request propagates as failure.

## Deterministic criteria
- **full-test-suite** in `/Users/shane/Projects/k-12Harness`
  - command: `.venv/bin/python -m pytest tests/ -q`
  - expected: The complete deterministic test suite exits with status 0 and reports no failed or unexpectedly passed tests.
  - machine-checked: exit code only
- **planner-focused-suite** in `/Users/shane/Projects/k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_planner.py tests/unit/test_intent_boundary.py -q`
  - expected: Both focused test modules exit with status 0, covering planner behaviour and pipeline propagation.
  - machine-checked: exit code only
- **school-resolution-checks** in `/Users/shane/Projects/k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_planner.py -q -k "school or identity or duplicate or conflict"`
  - expected: Selected tests pass and demonstrate unique resolution, clarification for zero or multiple matches, ordered deduplication, selector cardinality, conflict rejection, and no evidence fetch after failed preflight.
  - machine-checked: exit code only
- **input-policy-checks** in `/Users/shane/Projects/k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_planner.py tests/unit/test_intent_boundary.py -q -k "material or distance or clarification or default"`
  - expected: Selected tests pass and demonstrate policy-derived missing inputs, conditional distance_km materiality, stable clarification, omitted defaults, and clarification bypass of downstream answer stages.
  - machine-checked: exit code only
- **metric-and-request-checks** in `/Users/shane/Projects/k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_planner.py tests/unit/test_intent_boundary.py -q -k "metric or ratio or exact_request or normalization"`
  - expected: Selected tests pass and assert canonical passthrough, case and separator normalization, teaching-staff aliasing, stable deduplication, question-specific vocabularies, VC-POC-03 ratios routing, and exact request dictionaries for VC-POC-01, VC-POC-03, and VC-POC-04.
  - machine-checked: exit code only
- **error-semantics-checks** in `/Users/shane/Projects/k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_planner.py tests/unit/test_intent_boundary.py -q -k "bad_request or unanswerable or unknown or excluded or gated or data_unavailable or internal"`
  - expected: Selected tests pass; bad_request raises TerminalError with a retained invocation, while unknown, excluded, gated, unanswerable, data_unavailable, and internal preserve their specified outcomes.
  - machine-checked: exit code only

## Must not change
- The legacy PydanticAI agent-tool repair loop and its configured repair ceiling.
- Evidence-contract files, pinned hashes, request or response schemas, registry contents, and input-catalog snapshot.
- The evidence client protocol, deterministic data emitter, database access, or evidence calculation.
- Answer-view construction, narration, rendering, grounding, confidence, limitations, or claim-eligibility behaviour.
- Contract-owned exclusion wording or gated-question refusal wording.
- The prohibition on School_No to ACARA_SML_ID crosswalks.

## Failure behaviour
- **A named school returns no directory records.** → Return ClarificationNeeded identifying the unresolved school; do not mutate resolved IDs or call fetch_evidence.
- **A named school returns more than one directory record.** → Return ClarificationNeeded identifying the ambiguous school; do not choose the first candidate, mutate resolved IDs, or call fetch_evidence.
- **Resolved school identities disagree with explicit acara_sml_id or acara_sml_ids filters.** → Fail closed before fetch_evidence and retain no partial identity update in Deps.
- **Deduplication leaves fewer than two schools for VC-POC-04.** → Return clarification without evidence dispatch.
- **A requested selector cannot be normalized into the applicable static closed vocabulary.** → Fail before evidence dispatch and do not reinterpret the condition as an unsupported user question.
- **The emitter returns bad_request after deterministic planning.** → Append the failed fetch_evidence ToolInvocation and raise TerminalError containing the question ID and emitter error context.
- **The emitter returns unknown_question.** → Return the existing Unsupported not_in_registry outcome.
- **The emitter returns excluded_question.** → Return the existing contract-owned Unsupported outcome with its reason code and message.
- **The emitter returns unanswerable.** → Return the existing Unsupported unanswerable outcome without retry.
- **The emitter returns data_unavailable or internal.** → Raise TerminalError as an operational failure.

## Compatibility
- Existing excluded-topic, registry-owned exclusion, gated-question, and unknown-question branches retain their typed Unsupported outcomes and contract-owned text.
- Valid registered requests that are factually unanswerable retain the unanswerable abstention.
- Successful evidence envelopes continue to be returned unchanged and stored in deps.last_envelope.
- All school-directory and input-catalog preflights and evidence invocations remain observable in deps.tool_calls.
- The deterministic planner's TerminalError treatment of bad_request is intentionally distinct from the legacy model-driven tool loop, which may repair and retry.
- Release-derived and open vocabulary values remain type-and-shape constrained rather than being incorrectly enforced as static catalog enums.

## Compatibility baselines
- `contract/POC_CONTRACT_V0.3_SPEC.md` @ sha256:065f2a38b4befc09363b300cf7d601e4e782175ae6ef1ebca24d4eb173dfbe57 — Preserve the v0.3 input policy, error taxonomy, default ownership, and static-versus-release-derived vocabulary semantics.
- `contract/poc_question_registry.v0.3.0.json` @ sha256:3bea2093b2e900ecf7b27528aaad1dcec93f11791dd87ccf85c67f73f588b7cb — Use the registered input_policy, availability, selector constraints, metric lists, and preflight boundaries.
- `contract/question_schemas/v0.3.0/_index.json` @ sha256:e5b9877ffeb14e436433ebbc894de62a4bf8022324b9a33cfc80f4538d09bb1d — Preserve required, material, conditional-material, defaultable, optional, and gated classifications.
- `prompts/input_catalog.snapshot.json` @ sha256:a53480e98cca27a64518dbb940c6e83a69270132aa20b491874bb2b68f10fe10 — Preserve the captured canonical selector vocabularies.
- `prompts/rules.md` @ sha256:6a2a92d5ddba792b56c450ded4a2a982d04530b557967ab129c50fa37fb06d28 — Preserve smallest-request behaviour and the authorised teaching staff to teaching_fte interpretation.
- `evals/cases/golden_cases.json` @ sha256:10587743a8041e1d0520951e5b308a4776a2f462d0dae6f84a74cd9ce9632d92 — Preserve existing answer, clarification, refusal, gated, empty-success, and injection-resistance expectations except where this work item explicitly strengthens deterministic planning.

## Contract requirements
- **pinned-policy-authority** — Planner decisions must be derived from the pinned v0.3 registry, index, input-catalog mapping, and supplied request schemas rather than a new hand-maintained question policy.
- **policy-order** — Missing-input output must be stable across runs and ordered according to the pinned required, material, and applicable conditional-material declarations.
- **known-condition-only** — The only descriptive conditional-material expression implemented by this work item is distance_km when market_definition equals distance; any new condition form requires Definition Pack escalation.
- **no-invented-defaults** — The planner must not add values from input_policy.defaultable when absent; the emitter remains the owner of defaults and resolved_request.
- **preflight-boundary** — School-directory records establish current ACARA identity only and must not be used as campus, catchment, geography-crosswalk, or School_No evidence.
- **exact-schema-coverage** — Exact schema-shaped request assertions are mandatory for VC-POC-01, VC-POC-03, and VC-POC-04, the request schemas supplied in this Definition Pack.
- **direct-id-authority-conflict** — The golden case permits explicitly supplied ACARA ID 99999 to bypass directory lookup and yield an empty success, while the v0.3 specification classifies an ID absent from the pinned directory as bad_request. This work item must preserve explicit numeric IDs without new directory-membership validation until the Definition Pack resolves the conflict.
- **error-boundary-refinement** — Although the general v0.3 error table permits a model-driven harness to repair bad_request, this deterministic planner has no repair model; therefore a received bad_request is a planner or intent-boundary failure and must raise TerminalError rather than become Unsupported.

## Semantic criteria
- **no-model-missing-input-trust** — A VC-POC-05 intent without market_definition clarifies even when missing_material_inputs is empty, and an irrelevant model-reported missing field is ignored.
- **conditional-materiality** — A VC-POC-05 SA2 request does not ask for distance_km, while a distance request without distance_km does.
- **exact-school-identity** — Each school mention contributes an ID only after exactly one match; mention order is preserved, duplicates are removed, and partial resolution cannot leak into request or dependency state.
- **exact-metric-meaning** — For VC-POC-01, ["Total Enrolments", "teaching staff"] becomes ["total_enrolments", "teaching_fte"] without adding headcount or non-teaching metrics.
- **exact-ratio-routing** — A staffing-intensity selector for VC-POC-03 becomes an allowed ratios array, and the evidence request contains no metrics property.
- **exact-comparison-request** — A VC-POC-04 dispatch contains only ordered unique acara_sml_ids, explicitly requested canonical metrics, and any other explicitly supplied schema fields; year defaults remain omitted.
- **no-user-blame-for-planner-defect** — No deterministic planner defect, selector-shape defect, identity conflict, or emitter bad_request is rendered as an out-of-scope or otherwise Unsupported user question.

## Out of scope
- Generic local JSON Schema validation for every registered question when its request schema is not supplied in this Definition Pack.
- Support for conditional-material expressions beyond distance_km when market_definition equals distance.
- General natural-language synonym inference beyond mechanical case and separator normalization and aliases explicitly authorised by pinned rules.
- Directory-membership validation of caller-supplied numeric ACARA IDs while the golden-case and v0.3 specification conflict remains unresolved.
- Fuzzy school matching, candidate ranking, confidence scoring, or automatically selecting among ambiguous directory records.
- Changes to the legacy agent-tool bad_request repair-and-retry behaviour.
- Changes to the evidence contract, emitter, database, narration, answer view, rendering, graders, or presentation layer.
- New evidence capabilities, data sources, crosswalks, catchments, campus locations, transport, finance, capacity, outcomes, or forecasts.
