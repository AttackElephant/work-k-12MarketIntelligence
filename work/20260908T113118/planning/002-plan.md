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

- The v0.3 registry, `_index.json`, input-catalog schema, complete per-question request-schema set, and captured input-catalog snapshot are vendored and hash-pinned authorities.
- Required, material, conditional-material, defaultable, optional, and constraint declarations are distinct. Missing user-selectable required/material inputs clarify; unsupplied defaultable values remain omitted for emitter resolution.
- Named schools resolve only through the pinned directory. A directory record establishes identity only and cannot supply campus, catchment, geography-crosswalk, or School_No evidence. Zero or multiple matches are not safe selections.
- Static-closed selector vocabularies come deterministically from pinned contract artefacts. Use the registry's `metric_list` where declared and the input-catalog schema's `x-poc-request-field-to-catalog-list` plus embedded static enums for question/request-field routing, including VC-POC-03 `ratios` and VC-POC-11 `metrics` backed by `student_context_metrics`. Do not fetch these contract-static lists from the runtime emitter. Release-derived and open values remain shape-checked rather than enum-enforced.
- Every constructed request must validate against the registered pinned request schema before dispatch. This includes `additionalProperties: false`, nested filter keys, types, uniqueness/cardinality, conditional rules, and closed formats. The registry/emitter-owned VC-POC-09/10 explicit-filter constraint is an additional deterministic pre-dispatch rule because OD-07 deliberately leaves it outside the JSON Schema.
- `bad_request` disposition depends on origin. A planner-produced schema/key/shape defect is a harness failure. A user selector that cannot be normalized or a caller-supplied identity requiring correction is a clarification, not `Unsupported`. A well-typed release-derived/open non-match is a successful empty/partial result, never `bad_request`. After a request has passed deterministic schema and planner constraints, an emitter `bad_request` indicates planner/contract drift and raises `TerminalError`.
- `unknown_question`, `excluded_question`, and `unanswerable` remain typed abstentions. `data_unavailable` and `internal` remain operational failures. The legacy model-driven tool loop retains its separate repair ceiling.

## Current implementation

- `harness/planner.py` is implemented as Stage 2 of a four-stage pipeline, and `harness.agent.ask(deps, question)` invokes it. However, the shipping CLI does not currently select that path: `run.py::main()` calls `build_agent()` and passes the resulting agent to `converse()`, which calls `ask(agent, deps, ...)` and therefore uses the legacy PydanticAI tool loop. The deterministic planner is reachable in pipeline tests and direct callers but is not the default CLI production path.
- `harness/planner.py` asks for clarification only when the intent model has already populated `missing_material_inputs`; it does not independently compare assembled inputs with the pinned policy. Conditional materiality is widened by `contract.material_inputs_for()` and is not evaluated against the actual request, so the planner can miss a required radius or accept an irrelevant missing-input claim.
- School names are queried sequentially and the first returned record is accepted. Empty and ambiguous result sets do not stop evidence dispatch. Resolved IDs are accumulated directly in shared `Deps`, and explicit filter IDs can silently override directory-resolved IDs through `setdefault`.
- `requested_metrics` is copied verbatim into `metrics` for every question. Natural-language labels are not normalized, catalog membership is not checked, and VC-POC-03 receives `metrics` instead of its schema field `ratios`. Scope cleanup runs after construction and can leave an invalid or over-broad selector list.
- Requests are not validated locally against their registered per-question schemas. Unknown top-level or nested filter keys, wrong types, cardinality failures, and conditional failures reach the emitter. The permitted repository inventory contains the complete hash-pinned v0.3 request-schema set, so this is an implementation gap rather than a missing-definition constraint.
- VC-POC-09 and VC-POC-10 constraints requiring an explicit cohort-defining filter are not enforced by the planner. A missing, empty, or default-only definition can reach the emitter as `bad_request`.
- `_branch_evidence_error()` maps every emitter `bad_request` to `Unsupported(reason_kind="out_of_scope")`. This collapses planner-shape defects and caller-correctable identity/value problems into a user-facing refusal. Existing planner tests explicitly encode unsafe first-match resolution and this obsolete mapping.
- The legacy tool loop has a separate, intentional repair-once policy for `bad_request`. It remains exercised by replay and compatibility tests and is not changed by this work item.

## Proposed changes

- `harness/planner.py` and pinned contract access
  - Derive missing required, material, and applicable conditional-material inputs from `input_policy` and the assembled request, ignoring contradictory `StructuredIntent.missing_material_inputs`. Evaluate `distance_km` only when `market_definition == "distance"; do not change the legacy output gate or its strict O-8 xfail.
  - Treat school selector fields as satisfiable by explicit IDs or successful named-school resolution. Resolve each mention through the directory, require exactly one result, stage IDs locally in mention order, deduplicate without reordering, and commit to `Deps` only after all mentions succeed. Record every preflight.
  - Return `ClarificationNeeded` for zero/multiple school matches, insufficient distinct schools, absent user-selectable inputs, invalid caller-supplied identity reported by the emitter, and requested selectors that cannot be normalized. Use stable policy order, contract field names, and populated static allowed values where the pinned catalog mapping supplies them. Make no evidence call on these pre-dispatch clarification paths.
  - Raise `TerminalError` before dispatch for conflicts between explicit IDs and name-resolved identities, unknown fields introduced into the assembled request, and any other planner-produced request that cannot satisfy its registered schema. Never translate these defects to `Unsupported`.
  - Normalize selector labels by Unicode-safe case folding and separator normalization, preserve canonical values, apply only pinned semantic aliases (including `teaching staff` to `teaching_fte`), and deduplicate while preserving order. Apply the existing raw-question headcount/non-teaching scope rule before validation.
  - Derive selector routing from pinned artefacts: registry `metric_list` where present and the input-catalog schema's field-to-list mapping for `VC-POC-03.ratios`, grades, and other static selectors. Submit `metrics` for VC-POC-01/04/11 and `ratios` for VC-POC-03; omit selector fields when the user named none. Read static vocabularies from vendored pinned files and do not add a runtime input-catalog dependency.
  - Construct a fresh request from explicit intent fields and planner-owned selectors only. Validate it against the question's registered request schema before `fetch_evidence`, rejecting unknown top-level and nested properties, wrong types, bounds, uniqueness/cardinality, conditionals, and formats. Enforce VC-POC-09/10's registry-owned explicit-filter constraint before dispatch: absent, empty, or sector-default-only peer/segment definitions clarify; unknown filter keys are planner validation failures.
  - Record a failed evidence invocation before branching. If a schema-valid deterministic request still receives `bad_request`, distinguish the explicitly passed-through numeric-ID case and return identity clarification; otherwise raise `TerminalError` with question ID and emitter context because the result proves planner/contract drift. Preserve existing branches for excluded, gated, unknown, unanswerable, data-unavailable, and internal outcomes.

- `run.py`
  - Make the default CLI call `converse(None, deps, ...)`, selecting `ask(deps, question)` and the four-stage pipeline containing `plan_and_execute()`. Do not construct the legacy agent for this default path. Preserve credentials, emitter selection, rendering, exit codes, and clarification limits.
  - Retain explicit legacy-agent support in `converse()` for replay and compatibility tests; do not alter `harness/tools.py`, its `bad_request` repair ceiling, or its output semantics.

- Tests
  - Expand `tests/unit/test_planner.py` with exact typed assertions for unique, zero, ambiguous, duplicate, and conflicting school identities; transactional `Deps` updates; derived missing inputs; conditional distance; static allowed values; normalization/routing; schema validation; VC-POC-09/10 explicit and unknown-filter constraints; exact request dictionaries; and separated error origins.
  - Expand `tests/unit/test_intent_boundary.py` to prove planner clarification bypasses answer-view/narration, normalized schema-valid requests reach evidence, planner/schema defects propagate as `TerminalError`, and caller-correctable metric/identity problems remain clarification rather than `Unsupported`.
  - Update `tests/unit/test_run_cli.py` to assert the default CLI does not build or pass a legacy agent and reaches the pipeline signature. Keep existing legacy-loop tests unchanged.

## Verification approach

Run the complete deterministic suite:

```bash
.venv/bin/python -m pytest tests/ -q
```

Run the affected modules directly:

```bash
.venv/bin/python -m pytest tests/unit/test_planner.py tests/unit/test_intent_boundary.py tests/unit/test_run_cli.py -q
```

Use explicit node IDs for focused, non-vacuous checks; a missing or renamed test must fail collection:

```bash
.venv/bin/python -m pytest -q \n  tests/unit/test_planner.py::test_missing_market_definition_is_derived_without_model_hint \n  tests/unit/test_planner.py::test_distance_is_conditional_on_distance_market \n  tests/unit/test_planner.py::test_ambiguous_school_clarifies_without_fetch \n  tests/unit/test_planner.py::test_metric_normalization_and_exact_profile_request \n  tests/unit/test_planner.py::test_vc_poc_03_routes_selector_to_ratios \n  tests/unit/test_planner.py::test_unknown_request_property_fails_before_fetch \n  tests/unit/test_planner.py::test_peer_filters_require_explicit_nondefault_filter \n  tests/unit/test_planner.py::test_segment_filters_reject_unknown_nested_key \n  tests/unit/test_planner.py::test_schema_valid_emitter_bad_request_is_terminal \n  tests/unit/test_planner.py::test_invalid_explicit_identity_bad_request_clarifies \n  tests/unit/test_intent_boundary.py::test_planner_failure_is_not_unsupported \n  tests/unit/test_run_cli.py::test_main_uses_pipeline_path_by_default
```

Acceptance evidence must demonstrate: (1) stable required/material derivation independent of model missing-input claims; (2) conditional distance handling; (3) no evidence dispatch or partial `Deps` mutation after failed school resolution; (4) ordered unique scalar/array selector shapes; (5) canonical normalization and VC-POC-03 `ratios` routing; (6) validation against pinned schemas for all registered questions without enum-enforcing release-derived/open values; (7) VC-POC-09/10 absent/default-only filters clarify and unknown nested keys fail before dispatch; (8) planner defects and unexpected post-validation `bad_request` raise `TerminalError`, while unnormalizable user metrics and invalid explicit identities clarify; (9) exclusions, gating, unknown, unanswerable, empty/partial, data-unavailable, and internal retain their established outcomes; and (10) the default CLI executes the deterministic pipeline while legacy replay tests continue unchanged.

## Risks, assumptions and gaps

- The default CLI is confirmed to use the legacy agent because `run.py::main()` builds an agent and passes it to `converse()`. Routing that entry point through `converse(None, ...)` is the only scope expansion authorised here. Legacy replay/tool-loop behaviour remains unchanged and must continue to pass its existing tests.
- Stage 1 remains model-produced: deterministic planning means identical `StructuredIntent` values yield identical validation, clarification, resolution, and requests. This work does not re-parse raw user language, but it rejects or clarifies invalid extracted values instead of silently dispatching them. The existing scope guard intentionally still consults `deps.user_question` for headcount/non-teaching wording.
- The complete per-question schemas are present and pinned, so generic request validation is in scope. Use a validator supporting JSON Schema draft 2020-12 and resolve only registered paths beneath the pinned contract directory; schema-loading or reference-resolution failure is operational `TerminalError`.
- JSON Schema does not encode VC-POC-09/10's full explicit-filter rule by design (OD-07). Enforce that exact registry constraint in planner code and tests without generalising it into a new condition language.
- Conditional-material registry values remain descriptive strings. Implement only the known `distance_km` condition for VC-POC-05/06/07, bound to the exact pinned declaration; an unknown conditional expression is a contract-drift failure, not something to parse heuristically.
- No general synonym registry exists. Mechanical normalization and explicitly pinned aliases are allowed; an unnormalizable user selector returns `ClarificationNeeded` with the applicable static vocabulary and no evidence dispatch. It is not `Unsupported` or an operational failure.
- Caller-supplied numeric ACARA IDs remain exempt from a new directory preflight because the golden empty-success case and the specification's absent-directory `bad_request` rule conflict. If such an explicit ID alone produces emitter `bad_request`, return identity clarification. Other `bad_request` responses after local schema and constraint validation raise `TerminalError`; valid release-derived/open non-matches must continue as successful empty/partial envelopes.
- Directory search is per term in `EvidenceClient`; transactional resolution means concurrent or sequential per-name calls may be used, but all results are staged locally and committed together. It does not imply a new batch client protocol.
- The O-8 strict xfail belongs to the unchanged legacy output gate. The planner-level conditional fix must not modify `contract.material_inputs_for()`, `harness.tools`, or that test.

## Size estimate

- Scale: **medium**
- Files: 3
- Subsystems: deterministic planner, school-directory preflight handling, input-catalog normalization, pipeline error propagation, unit/integration tests
- Rough change: approximately 150–250 implementation lines plus 200–300 lines of focused deterministic tests.
## Planning gaps

See Risks, assumptions and gaps above; no additional gaps are asserted by this mechanical completion.
## Amendment trace

```json
{
  "base_plan": {
    "artifact_id": "A4",
    "events": [
      {
        "seq": 72,
        "sha256": "fb92d0a1299f1f8f2ec81c61b30fa893c4a432363bf4adcf5bd30d4f2a122303"
      }
    ],
    "path": "planning/001-plan.md",
    "sha256": "44525a85c16a4756c9115545bbce79a5351605ec1bbb5c974af1d25eecc86489"
  },
  "basis_sha256": "e89d9fb430662b4b0ddfd24287f75907cffe0adc7f7ffd5943c80557a00faaaf",
  "retained_rule": "Unchallenged content is retained byte-for-byte and accepted by default.",
  "retained_sections": [
    "Bounded objective",
    "Size estimate"
  ],
  "changes": {
    "basis_sha256": "e89d9fb430662b4b0ddfd24287f75907cffe0adc7f7ffd5943c80557a00faaaf",
    "findings": [
      {
        "id": "production-path",
        "status": "resolved",
        "resolution": "The default CLI currently constructs an agent and passes it to converse(), which selects the legacy PydanticAI tool loop; plan_and_execute() is reachable only through ask(deps, question). The amendment therefore includes the minimal production-wiring change needed to make the deterministic pipeline the default CLI path while retaining the legacy path for scripted compatibility tests."
      },
      {
        "id": "bad-request-origin",
        "status": "resolved",
        "resolution": "The amendment separates pre-dispatch planner/schema defects, unnormalizable user selectors, caller-supplied directory identities, release-derived/open non-matches, and post-validation emitter bad_request. Only planner defects and an unexpected bad_request after successful deterministic validation are operational failures; user-correctable selector or identity problems produce typed clarification, and valid release-derived/open non-matches remain successful empty/partial results."
      },
      {
        "id": "request-validation",
        "status": "resolved",
        "resolution": "All per-question v0.3 request schemas are present in the permitted, hash-pinned repository inventory. The amended plan validates every constructed request against its registered pinned schema before evidence dispatch, including additionalProperties, nested keys, types, cardinality, conditionals, and formats."
      },
      {
        "id": "static-vocabulary",
        "status": "resolved",
        "resolution": "Static normalization uses vendored pinned authorities: the registry metric_list and the input-catalog schema's question-field mapping and static enums, with the captured snapshot only as a consistency baseline. It does not call the runtime emitter for contract-static vocabularies."
      },
      {
        "id": "typed-outcomes",
        "status": "resolved",
        "resolution": "Zero or multiple directory matches and user selectors that cannot be normalized return ClarificationNeeded with stable contract field names and no evidence dispatch. Conflicting explicit and name-resolved identities, unknown request keys introduced by planning, and other impossible planner-produced shapes raise TerminalError. None becomes Unsupported."
      },
      {
        "id": "filter-constraints",
        "status": "resolved",
        "resolution": "Verification now explicitly covers VC-POC-09 and VC-POC-10 nested unknown-filter rejection and the registry/emitter rule requiring at least one explicit cohort-defining filter, with clarification before dispatch for absent or default-only definitions."
      },
      {
        "id": "nonvacuous-tests",
        "status": "resolved",
        "resolution": "Targeted verification uses explicit pytest node IDs rather than -k substring selection, so a renamed or absent test causes collection failure instead of a vacuous successful run."
      },
      {
        "id": "conditional-scope",
        "status": "resolved",
        "resolution": "Scope expands only because production-path inspection proved the CLI defaults to the legacy loop. The amendment adds the minimal run.py wiring and CLI regression test needed to route the shipping CLI through the deterministic pipeline; the legacy loop itself remains unchanged."
      }
    ],
    "replacements": [
      {
        "section": "Current implementation",
        "findings": [
          "production-path"
        ],
        "dependency_reason": null,
        "section_sha256": "7afd25099a315ad83c5daba11c386c1150e4af2947ff238c202d90fae7e0fc02"
      },
      {
        "section": "Definition Pack interpretation",
        "findings": [
          "bad-request-origin"
        ],
        "dependency_reason": null,
        "section_sha256": "9995e644f893590e7c2ec44140dafaa6c6ed87df9d8eb63cada34ee61c6f3dac"
      },
      {
        "section": "Proposed changes",
        "findings": [
          "request-validation",
          "static-vocabulary",
          "typed-outcomes"
        ],
        "dependency_reason": null,
        "section_sha256": "4d4595373388175559c150286cef30dd371f489566d7b65836825467d8809d06"
      },
      {
        "section": "Verification approach",
        "findings": [
          "filter-constraints",
          "nonvacuous-tests"
        ],
        "dependency_reason": null,
        "section_sha256": "e437f02ce2523bc2b3c2c4636cd87ddf00281e9cdb74fa930c158f05ef65542d"
      },
      {
        "section": "Risks, assumptions and gaps",
        "findings": [
          "conditional-scope"
        ],
        "dependency_reason": null,
        "section_sha256": "bff66893e5bf1ead9be48a0876c79e31cd09dda3d94df70ca4edb93d39b9f786"
      }
    ]
  },
  "finding_trace": [
    {
      "finding": "production-path",
      "source_evidence": [
        "challenge/005-problem-analysis-v3.md",
        "capture/imports/011-critic-009-plan-review.log"
      ],
      "affected_plan_sections": [
        "Current implementation"
      ],
      "resolution": "The default CLI currently constructs an agent and passes it to converse(), which selects the legacy PydanticAI tool loop; plan_and_execute() is reachable only through ask(deps, question). The amendment therefore includes the minimal production-wiring change needed to make the deterministic pipeline the default CLI path while retaining the legacy path for scripted compatibility tests.",
      "necessary_dependent_changes": [],
      "retained_sections": [
        "Bounded objective",
        "Size estimate"
      ]
    },
    {
      "finding": "bad-request-origin",
      "source_evidence": [
        "challenge/005-problem-analysis-v3.md",
        "capture/imports/011-critic-009-plan-review.log"
      ],
      "affected_plan_sections": [
        "Definition Pack interpretation"
      ],
      "resolution": "The amendment separates pre-dispatch planner/schema defects, unnormalizable user selectors, caller-supplied directory identities, release-derived/open non-matches, and post-validation emitter bad_request. Only planner defects and an unexpected bad_request after successful deterministic validation are operational failures; user-correctable selector or identity problems produce typed clarification, and valid release-derived/open non-matches remain successful empty/partial results.",
      "necessary_dependent_changes": [],
      "retained_sections": [
        "Bounded objective",
        "Size estimate"
      ]
    },
    {
      "finding": "request-validation",
      "source_evidence": [
        "challenge/005-problem-analysis-v3.md",
        "capture/imports/011-critic-009-plan-review.log"
      ],
      "affected_plan_sections": [
        "Proposed changes"
      ],
      "resolution": "All per-question v0.3 request schemas are present in the permitted, hash-pinned repository inventory. The amended plan validates every constructed request against its registered pinned schema before evidence dispatch, including additionalProperties, nested keys, types, cardinality, conditionals, and formats.",
      "necessary_dependent_changes": [],
      "retained_sections": [
        "Bounded objective",
        "Size estimate"
      ]
    },
    {
      "finding": "static-vocabulary",
      "source_evidence": [
        "challenge/005-problem-analysis-v3.md",
        "capture/imports/011-critic-009-plan-review.log"
      ],
      "affected_plan_sections": [
        "Proposed changes"
      ],
      "resolution": "Static normalization uses vendored pinned authorities: the registry metric_list and the input-catalog schema's question-field mapping and static enums, with the captured snapshot only as a consistency baseline. It does not call the runtime emitter for contract-static vocabularies.",
      "necessary_dependent_changes": [],
      "retained_sections": [
        "Bounded objective",
        "Size estimate"
      ]
    },
    {
      "finding": "typed-outcomes",
      "source_evidence": [
        "challenge/005-problem-analysis-v3.md",
        "capture/imports/011-critic-009-plan-review.log"
      ],
      "affected_plan_sections": [
        "Proposed changes"
      ],
      "resolution": "Zero or multiple directory matches and user selectors that cannot be normalized return ClarificationNeeded with stable contract field names and no evidence dispatch. Conflicting explicit and name-resolved identities, unknown request keys introduced by planning, and other impossible planner-produced shapes raise TerminalError. None becomes Unsupported.",
      "necessary_dependent_changes": [],
      "retained_sections": [
        "Bounded objective",
        "Size estimate"
      ]
    },
    {
      "finding": "filter-constraints",
      "source_evidence": [
        "challenge/005-problem-analysis-v3.md",
        "capture/imports/011-critic-009-plan-review.log"
      ],
      "affected_plan_sections": [
        "Verification approach"
      ],
      "resolution": "Verification now explicitly covers VC-POC-09 and VC-POC-10 nested unknown-filter rejection and the registry/emitter rule requiring at least one explicit cohort-defining filter, with clarification before dispatch for absent or default-only definitions.",
      "necessary_dependent_changes": [],
      "retained_sections": [
        "Bounded objective",
        "Size estimate"
      ]
    },
    {
      "finding": "nonvacuous-tests",
      "source_evidence": [
        "challenge/005-problem-analysis-v3.md",
        "capture/imports/011-critic-009-plan-review.log"
      ],
      "affected_plan_sections": [
        "Verification approach"
      ],
      "resolution": "Targeted verification uses explicit pytest node IDs rather than -k substring selection, so a renamed or absent test causes collection failure instead of a vacuous successful run.",
      "necessary_dependent_changes": [],
      "retained_sections": [
        "Bounded objective",
        "Size estimate"
      ]
    },
    {
      "finding": "conditional-scope",
      "source_evidence": [
        "challenge/005-problem-analysis-v3.md",
        "capture/imports/011-critic-009-plan-review.log"
      ],
      "affected_plan_sections": [
        "Risks, assumptions and gaps"
      ],
      "resolution": "Scope expands only because production-path inspection proved the CLI defaults to the legacy loop. The amendment adds the minimal run.py wiring and CLI regression test needed to route the shipping CLI through the deterministic pipeline; the legacy loop itself remains unchanged.",
      "necessary_dependent_changes": [],
      "retained_sections": [
        "Bounded objective",
        "Size estimate"
      ]
    }
  ]
}
```
