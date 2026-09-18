<!-- Derived view of 003-contract.yaml. The YAML is authoritative. Regenerate; never edit. sha256=2b7a5d5eecf211e61e3b45e11f2addf1fa58f159c1e9cda86960dc73863139df -->

# Acceptance contract — work item 20260908T113118

contract_version: 0.1.0  ·  artifact_version: 1

## Observable behaviour
- **planner-derived-inputs** — The deterministic planner derives absent required, material, and applicable conditional-material inputs from the pinned question policy and assembled request, independently of StructuredIntent.missing_material_inputs.
- **conditional-distance** — For VC-POC-05, VC-POC-06, and VC-POC-07, distance_km is required only when market_definition is distance.
- **safe-transactional-school-resolution** — Each named school must resolve to exactly one directory record; IDs are staged in mention order, deduplicated without reordering, and committed to Deps only after all mentions resolve.
- **exact-selector-shapes** — Single-school questions submit one acara_sml_id; multi-school questions submit ordered unique acara_sml_ids; VC-POC-04 requires at least two distinct IDs.
- **metric-normalization-routing** — Requested selectors are Unicode-safe case-folded and separator-normalized, canonical values and pinned aliases are accepted, order-preserving deduplication is applied, and selectors are routed to the question-specific field and static vocabulary.
- **smallest-schema-valid-request** — The planner constructs a fresh request from explicit intent values and the applicable planner-owned selectors, omits unsupplied defaults, and validates it against the registered pinned request schema before evidence dispatch.
- **explicit-filter-constraints** — VC-POC-09 and VC-POC-10 require an explicit cohort-defining filter: absent, empty, or sector-default-only definitions clarify, while unknown nested keys fail before dispatch.
- **bad-request-origin-separation** — Unnormalizable user selectors and invalid caller-supplied numeric identities clarify; planner/schema defects and unexpected bad_request responses after deterministic validation raise TerminalError; none becomes Unsupported.
- **default-cli-pipeline** — The default CLI invokes converse with no legacy agent so that ask(deps, question) executes the four-stage deterministic pipeline, while explicit legacy-agent callers remain supported.

## Required outputs
- `harness/planner.py` — Deterministic policy derivation, transactional identity resolution, static selector normalization and routing, pinned-schema validation, exact request construction, explicit-filter enforcement, and origin-aware error handling.
- `run.py` — Default CLI wiring selects the deterministic pipeline without constructing or passing a legacy agent.
- `tests/unit/test_planner.py` — Focused coverage of input derivation, school resolution, selector normalization, all-question schema validation, explicit-filter constraints, exact requests, and error-origin semantics.
- `tests/unit/test_intent_boundary.py` — Pipeline coverage proving clarification bypasses narration, validated requests reach evidence, and planner failures propagate without becoming Unsupported.
- `tests/unit/test_run_cli.py` — CLI coverage proving the default path does not build or pass a legacy agent and selects the deterministic pipeline signature.

## Deterministic criteria
- **full-test-suite** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/ -q`
  - expected: The complete deterministic suite exits zero with no failures and no unexpected passes.
  - machine-checked: yes
- **affected-modules** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_planner.py tests/unit/test_intent_boundary.py tests/unit/test_run_cli.py -q`
  - expected: All affected test modules collect and pass.
  - machine-checked: yes
- **focused-nonvacuous-acceptance** in `k-12Harness`
  - command: `.venv/bin/python -m pytest -q tests/unit/test_planner.py::test_missing_market_definition_is_derived_without_model_hint tests/unit/test_planner.py::test_distance_is_conditional_on_distance_market tests/unit/test_planner.py::test_ambiguous_school_clarifies_without_fetch tests/unit/test_planner.py::test_metric_normalization_and_exact_profile_request tests/unit/test_planner.py::test_vc_poc_03_routes_selector_to_ratios tests/unit/test_planner.py::test_unknown_request_property_fails_before_fetch tests/unit/test_planner.py::test_peer_filters_require_explicit_nondefault_filter tests/unit/test_planner.py::test_segment_filters_reject_unknown_nested_key tests/unit/test_planner.py::test_schema_valid_emitter_bad_request_is_terminal tests/unit/test_planner.py::test_invalid_explicit_identity_bad_request_clarifies tests/unit/test_intent_boundary.py::test_planner_failure_is_not_unsupported tests/unit/test_run_cli.py::test_main_uses_pipeline_path_by_default`
  - expected: Every named acceptance test exists, collects, and passes; a missing or renamed test makes the command fail.
  - machine-checked: yes

## Must not change
- The legacy PydanticAI agent-tool repair loop, its repair ceiling, and its replay compatibility behaviour.
- Evidence-contract files, PINNED.json, registry contents, request or response schemas, and the captured input-catalog snapshot.
- The evidence client protocol, runtime emitter, database access, or evidence calculations.
- Answer-view, narration, rendering, grounding, confidence, limitations, claim eligibility, or contract-owned refusal wording.
- The prohibition on School_No to ACARA_SML_ID crosswalks.
- contract.material_inputs_for() and the unchanged legacy O-8 strict xfail.

## Failure behaviour
- **A named school has zero or multiple directory matches.** → Return ClarificationNeeded for that school, record the preflight, make no evidence call, and retain no partial resolved-ID mutation.
- **Explicit school IDs conflict with name-resolved identities.** → Raise TerminalError before evidence dispatch and retain no partial identity update.
- **Deduplication leaves fewer than two identities for VC-POC-04.** → Return ClarificationNeeded without evidence dispatch.
- **A required or applicable material user-selectable field is absent.** → Return ClarificationNeeded in stable pinned-policy order, with contract field names and static allowed values where available, without evidence dispatch.
- **A requested static selector cannot be normalized to the applicable pinned vocabulary.** → Return ClarificationNeeded with the applicable static vocabulary and make no evidence call.
- **A planner-built request violates its registered schema, contains an unknown top-level or nested field, or cannot load or resolve its pinned schema safely.** → Raise TerminalError before evidence dispatch; do not return Unsupported.
- **VC-POC-09 or VC-POC-10 lacks an explicit nondefault cohort-defining filter.** → Return ClarificationNeeded without evidence dispatch.
- **A locally validated request containing only a caller-supplied numeric ACARA identity receives bad_request.** → Record the failed fetch invocation and return identity clarification.
- **Any other locally validated deterministic request receives bad_request.** → Record the failed fetch invocation and raise TerminalError containing the question ID and emitter context.
- **The emitter returns unknown_question, excluded_question, or unanswerable.** → Preserve the established typed abstention and contract-owned reason semantics without retry.
- **The emitter returns data_unavailable or internal.** → Raise TerminalError as an operational failure.

## Compatibility
- Excluded-topic, registry-owned exclusion, gated-question, unknown-question, and unanswerable outcomes retain their established typed outcomes and contract-owned text.
- Successful complete, partial, and empty envelopes remain successful and are stored unchanged in deps.last_envelope.
- Well-typed release-derived and open values that match no facts remain successful empty or partial results, not schema failures.
- Every directory preflight and evidence invocation, including failed invocations, remains observable in deps.tool_calls.
- The deterministic planner's TerminalError handling remains distinct from the legacy model-driven repair-once loop.
- converse() continues to accept an explicit legacy agent for replay and compatibility tests.
- CLI credential selection, emitter selection, rendering, clarification limit, and exit-code behaviour remain unchanged.

## Compatibility baselines
- `contract/POC_CONTRACT_V0.3_SPEC.md` @ sha256:065f2a38b4befc09363b300cf7d601e4e782175ae6ef1ebca24d4eb173dfbe57 — Preserve v0.3 request validation, default ownership, error taxonomy, and static-versus-release-derived semantics.
- `contract/poc_question_registry.v0.3.0.json` @ sha256:3bea2093b2e900ecf7b27528aaad1dcec93f11791dd87ccf85c67f73f588b7cb — Use registered input policies, request-schema references, availability rules, metric lists, and VC-POC-09/10 constraints.
- `contract/question_schemas/v0.3.0/_index.json` @ sha256:e5b9877ffeb14e436433ebbc894de62a4bf8022324b9a33cfc80f4538d09bb1d — Preserve required, material, conditional-material, defaultable, optional, support, and gating classifications.
- `contract/poc_input_catalog.v0.3.0.schema.json` @ sha256:41bad353798aa0c2dbbd2fcc33db806430a12c371db8c20ef35d822aac207dc2 — Use its question-field mapping and embedded static enums without enum-enforcing release-derived or open values.
- `prompts/input_catalog.snapshot.json` @ sha256:a53480e98cca27a64518dbb940c6e83a69270132aa20b491874bb2b68f10fe10 — Retain consistency with captured canonical vocabularies without adding a runtime catalog dependency.
- `prompts/rules.md` @ sha256:6a2a92d5ddba792b56c450ded4a2a982d04530b557967ab129c50fa37fb06d28 — Preserve smallest-request behaviour and the pinned teaching staff to teaching_fte alias.
- `evals/cases/golden_cases.json` @ sha256:10587743a8041e1d0520951e5b308a4776a2f462d0dae6f84a74cd9ce9632d92 — Preserve existing answer, clarification, refusal, gated, empty-success, and injection-resistance expectations.

## Contract requirements
- **manifest-allowlist-conflict** — The amended plan requires run.py and tests/unit/test_run_cli.py, but manifest scope.declared_files lists only harness/planner.py, tests/unit/test_intent_boundary.py, and tests/unit/test_planner.py. Execution must not begin until the human reviewer expands the manifest diff allowlist to include the two additional required outputs.
- **pinned-policy-authority** — Planner policy and schema routing must come from hash-pinned v0.3 artefacts, not a new hand-maintained per-question policy.
- **complete-schema-validation** — Every constructed request for every registered question must validate against its registered pinned Draft 2020-12 request schema before dispatch, including additionalProperties, nested keys, types, bounds, uniqueness, cardinality, conditionals, and formats.
- **safe-schema-resolution** — Only registered request-schema paths beneath the pinned contract directory may be loaded; load or reference-resolution failures are TerminalError.
- **policy-order** — Clarification fields are stable across runs and follow pinned required, material, then applicable conditional-material order.
- **known-condition-only** — Only the exact pinned distance_km condition for VC-POC-05/06/07 is interpreted; an unknown conditional expression is contract drift and raises TerminalError.
- **no-invented-defaults** — Unsupplied defaultable values are omitted so the emitter remains owner of defaults and resolved_request.
- **preflight-boundary** — A school-directory record establishes identity only and supplies no campus, catchment, geography-crosswalk, or School_No evidence.
- **static-vocabulary-source** — Static selector routing uses registry metric_list and the input-catalog schema's question-field mapping and embedded enums; it does not call the runtime emitter for contract-static lists.
- **direct-id-authority-conflict** — golden_cases.json permits explicit ACARA ID 99999 to bypass directory lookup and yield empty success, while the v0.3 specification says an ID absent from the pinned directory is bad_request. Preserve the plan's bounded resolution: do not add directory preflight for caller-supplied numeric IDs, and treat a resulting bad_request as identity clarification until the Definition Pack owner resolves the conflict.
- **od07-explicit-filter-rule** — Enforce the registry-owned VC-POC-09/10 explicit-filter constraint in addition to JSON Schema without creating a general condition language.
- **invocation-recording** — Record every completed directory preflight and every evidence attempt, including failed attempts, before returning or raising.

## Semantic criteria
- **no-model-missing-input-trust** — VC-POC-05 without market_definition clarifies even when the model reports no missing input, and irrelevant model-reported missing fields are ignored.
- **conditional-materiality** — An SA2 market request does not ask for distance_km, while a distance market without distance_km does.
- **transactional-identity** — Each school contributes an ID only after exactly one match; mention order is preserved, duplicates are removed, and a later failed match leaks no partial state or request.
- **exact-profile-normalization** — For VC-POC-01, Total Enrolments and teaching staff become total_enrolments and teaching_fte without adding headcount or non-teaching metrics.
- **exact-ratio-routing** — VC-POC-03 staffing-intensity selectors use the ratios vocabulary and ratios request field, with no metrics property.
- **exact-comparison-request** — VC-POC-04 dispatch contains ordered unique IDs, explicitly requested canonical metrics, and only other explicitly supplied schema fields; emitter-owned year defaults remain omitted.
- **schema-and-filter-fail-closed** — Unknown top-level or nested properties, invalid shapes, and impossible planner-produced requests fail before dispatch; VC-POC-09/10 missing or default-only cohort definitions clarify.
- **bad-request-origin** — Caller-correctable identity problems clarify, while planner defects and unexpected post-validation bad_request responses fail operationally; neither is rendered as an unsupported question.
- **production-routing** — run.py main does not call build_agent for its default execution and passes None to converse, while tests that pass an explicit agent continue to use the legacy path.

## Out of scope
- Re-parsing raw natural language in Stage 2; determinism is defined for identical StructuredIntent values.
- Conditional-material expressions beyond the exact pinned distance_km condition.
- General synonym inference beyond mechanical normalization and explicitly pinned aliases.
- Directory-membership preflight for caller-supplied numeric ACARA IDs while the Definition Pack conflict remains unresolved.
- Fuzzy school matching, ranking, match confidence, or selecting among ambiguous records.
- A new batch directory protocol.
- Changes to the legacy agent-tool repair loop or its output semantics.
- Changes to contract artefacts, the emitter, database, answer view, narration, rendering, graders, or presentation semantics.
- New evidence capabilities, sources, crosswalks, catchments, campus locations, transport, finance, capacity, outcomes, or forecast execution.
