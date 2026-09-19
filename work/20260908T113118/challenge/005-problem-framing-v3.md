# Independent Requirements and Problem Framing

*Provisional framing only. This stage has no repository-implementation evidence: `harness/planner.py`, its tests, the implementation plan, and the acceptance contract have not been read. Every statement about "the planner" below is a requirement or a hypothesis for a later current-state inspection to test, not a finding about current behaviour. Framing is drawn from the pinned v0.3.0 Definition Pack now provided: `request.md`, `PINNED.json`, `POC_CONTRACT_V0.3_SPEC.md`, `POC_EVIDENCE_CONTRACT.md`, the error/input-catalog/school-directory schemas, `poc_question_registry.v0.3.0.json`, the VC-POC-01/03/04 request schemas, `_index.json`, `docs/decisions.md`, `golden_cases.json`, `input_catalog.snapshot.json`, and `rules.md`.*

## Authoritative requirements

The request names five obligations for `harness/planner.py`. Anchored to the pinned Definition Pack they decompose as follows.

1. **Deterministically derive required inputs.** For a chosen question ID, the planner must determine — from contract data (the registry `input_policy`, the per-question request schema `required`/`x-poc`, and `_index.json`) — which inputs are `required`, `material`, `conditional_material`, `defaultable`, or `optional`. "Deterministically" means: same intent + same evidence in ⇒ same derivation, with no model-invented slots and no reliance on nondeterministic iteration. Material-but-missing inputs must drive a clarification (`rules.md` step 3); defaultable inputs must be omitted so the data layer defaults them and echoes them in `resolved_request`.
2. **Resolve schools safely.** Named-school questions must resolve identity only via the `--school-directory` preflight (`search_schools`), never by matching a name against any other field or fabricating an `acara_sml_id` (ADR-001; `rules.md` step 1). Ambiguity (multiple matches) must clarify rather than pick; multiple named schools should resolve in one directory call; a directory record must not be treated as a location/`School_No`/catchment fact.
3. **Normalize requested metrics.** User language must map onto the correct static-closed vocabulary for the question's metric list — `profile_metrics` (VC-POC-01/04), `student_context_metrics` (VC-POC-11), `ratios` (VC-POC-03), `grades` (VC-POC-02/07) — honouring the `rules.md` disambiguations ("teaching staff" ⇒ `teaching_fte`; headcount only on explicit request; non-teaching only on explicit request), producing the *smallest* set that answers the words used. If no metric is named, `metrics` is omitted so defaults apply.
4. **Construct exact evidence requests.** The emitted structured request must validate against the per-question request schema, which sets `additionalProperties: false` on the request object and every nested filter object. It must carry required fields, omit defaultables it does not set, and contain no extraneous/unknown fields. The eventual `resolved_request` and evidence filters must be able to agree with it.
5. **Surface planner-generated bad requests as failures, not unsupported user questions.** When the emitter rejects a request the *planner itself constructed* as `bad_request` (shape/vocabulary/cardinality/conditional defect), that must surface as a harness/planner failure signal — distinct from a legitimate `Unsupported` refusal (excluded topic) or a `ClarificationNeeded`. The anti-pattern to prevent is masking a planner defect as a user-facing "we can't answer that."

Cross-cutting contract obligations any solution inherits: the harness computes nothing and every number originates in the envelope (ADR-001); the contract is pinned by hash at version 0.3.0 (ADR-008, `PINNED.json`); the error taxonomy is `bad_request | unknown_question | excluded_question | unanswerable | data_unavailable | internal` (error schema/registry); a refusal/clarification is a correct outcome, not a failure (`rules.md`).

## Scope and compatibility boundaries

- **Contract-owned, not planner-owned.** Input policy, vocabularies, defaults, materiality, grade, confidence, limitations, and refusal text belong to the pinned contract. The planner selects and slot-fills; it must not re-derive, hardcode, or paraphrase these. Any hand-maintained materiality/condition table risks drift and contradicts the O-8 warning against "a second hand-maintained condition table."
- **Static-closed vs release-derived vs open vocabularies (OD-03, input-catalog `x-poc-static-vs-release-derived`).** Static-closed violations (metrics, grades, ratios, `market_definition`, SEIFA/`erp_sex`, filter keys) are `bad_request`. Release-derived values (`acara_sml_id`, `sa2_codes`, sector/type/etc. *values*, `erp_age`, `age_bands`) and open sets (`suburbs`, `postcodes`) are shape-constrained only: a well-typed non-matching value must yield an empty/partial result, **never** `bad_request` and never a refusal. The planner must not treat these as invalid.
- **Identity error boundary.** An `acara_sml_id` absent from the pinned directory is a *repairable* `bad_request` (selector obligation); a directory-valid record lacking geography/coordinate is `unanswerable`; `zone_entity_code` absent-but-well-formed is `unanswerable` (OD-05). These are user/data conditions, distinct from the planner constructing a structurally malformed request.
- **Conditional materiality (`distance_km`).** Material only when `market_definition == distance` for VC-POC-05/06/07 (registry `conditional_material`; `_index.json`; golden `clarify/distance_km-conditional`). O-8 pins a strict `xfail` because the legacy gate treats it as unconditional; G-4 records that the condition was extracted by string-splitting registry prose. The v0.3.0 per-question schemas' `x-poc.conditional_material` are the sanctioned source; whether this work item is expected to consume them (CR-5 direction) or preserve the shim is a boundary the later inspection must confirm.
- **Emitter alignment.** OD-07: "at least one explicit filter" for VC-POC-09/10 stays registry/emitter-enforced (schemas express only `minProperties >= 1`); VC-POC-09 defaults `peer_filters.sectors` to `[Catholic, Independent]` and requires one filter beyond that default. VC-POC-14 is gated (`excluded_question`, `not_yet_validated`) and must route through the registry-owned refusal, never the planner-failure path.
- **Out of scope.** No SQL/DB access, no arithmetic, no `School_No`↔`ACARA_SML_ID` crosswalk, no name-matching across systems, no rendering/confidence-grade assignment (ADR-009). Retry ownership and the repair cap live elsewhere (ADR-003; ADR-014 `RepairCapExceeded`); the planner interacts with those boundaries but should not duplicate them.

## Initial hypotheses to test

*(Each provisional; confirm against the actual `harness/planner.py` and its tests.)*

- **H1 — Contract-sourced derivation.** Required/material/defaultable inputs are derived from the pinned artifacts (registry/schema/`_index.json`), not a hardcoded list such as the `settings.MATERIAL_INPUTS` shim flagged for deletion (O-1).
- **H2 — Material vs defaultable split.** Missing *material* inputs produce `ClarificationNeeded`; *defaultable* inputs are omitted and left to the emitter, matching the `should_clarify` vs `should_answer` split in `golden_cases.json`.
- **H3 — Safe school resolution.** Resolution goes through the directory preflight, never name-matches, clarifies on multiple matches, batches multiple named schools into one call, and does not fabricate IDs.
- **H4 — Metric normalization fidelity.** NL maps to the correct per-question metric list and vocabulary, applies the `teaching_fte`/headcount/non-teaching disambiguations, and emits the smallest request (no related-but-unrequested metrics).
- **H5 — Schema-exact construction.** Constructed requests contain only permitted fields (`additionalProperties: false` at request and nested-filter level) and match golden `expected_request` shapes (respecting `unordered_request_keys`).
- **H6 — Bad-request-as-failure classification.** A planner-constructed `bad_request` surfaces as a failure signal distinct from (a) a legitimate excluded/`Unsupported` refusal, (b) a `ClarificationNeeded`, and (c) a user-identity `bad_request` that is legitimately repaired/retried once.
- **H7 — Determinism.** Identical intent + evidence yield byte-identical requests (no set/dict-order nondeterminism), consistent with replay-equality expectations.
- **H8 — Conditional-materiality treatment.** `distance_km` is material only under `market_definition == distance`, and the mechanism does not depend on string-splitting registry prose (G-4/O-8).
- **H9 — Release-derived tolerance.** Well-typed but non-matching release-derived/open values are passed through (to become empty/partial), not rejected as `bad_request` or refused (OD-03).

## Required implementation and test evidence

A later inspection should look for, and acceptance should demand, evidence of:

- **Derivation logic** in `harness/planner.py` that reads input policy from the pinned contract (not a duplicated table), covering `required`, `material`, `conditional_material`, `defaultable`, `optional` for every harness-safe question.
- **School-resolution path** demonstrating directory-preflight use, ambiguity ⇒ clarify, multi-name batching, and refusal to name-match or synthesize IDs.
- **Metric-normalization unit tests** exercising: correct metric list per question; `teaching_fte` vs `teaching_staff_headcount`; non-teaching only on explicit ask; smallest-set behaviour; omission ⇒ defaults.
- **Request-construction tests** asserting schema validity, `additionalProperties: false` compliance (no extra fields), and exact match to `golden_cases.json` `expected_request` (respecting `unordered_request_keys`).
- **Error-classification tests** separating a *planner-generated* `bad_request` (failure) from a user-identity `bad_request` (repairable/retry), an excluded-topic `Unsupported`, an `unanswerable`, and a `ClarificationNeeded` — including a negative test that a planner defect is NOT emitted to the user as `Unsupported`.
- **Determinism/replay evidence** (repeat-run equality of constructed requests).
- **Conditional-materiality tests** for VC-POC-05/06/07 `distance_km` in both distance and non-distance cases, ideally retiring the O-8 `xfail` cleanly rather than adding a parallel condition table.
- **Contract-pin integrity** unaffected (ADR-008): no edits to `PINNED.json` to make tests pass; alignment with registry version 0.3.0.
- **Golden-suite alignment** across `should_answer`, `should_clarify`, `should_refuse`, and `should_resist_injection` directions, including the empty/gappy VC-POC-01 cases and the gated VC-POC-14 refusal.

## Failure modes and acceptance hazards

- **Masking the target defect.** The central hazard: a planner defect that produces an invalid request being silently converted into a user-facing `Unsupported`/refusal. Tests that only assert "a refusal occurred" would pass while hiding exactly the bug the request targets. Evidence must positively distinguish failure from refusal.
- **Materiality mis-gating.** Over-defaulting a genuinely material input (skips a needed clarify → wrong/underspecified answer) or over-clarifying a defaultable input (wasted round-trip; the known `distance_km`-on-SA2 O-8 mode). Only the former corrupts an answer; both are acceptance concerns.
- **Name-matching leakage.** Any resolution path that matches names against non-directory fields, or fabricates/guesses `acara_sml_id`, violates ADR-001 and the directory boundary.
- **Metric superset / grounding risk.** Adding related-but-unrequested metrics inflates the request and endangers grounding (numbers with no user referent).
- **Vocabulary misclassification.** Rejecting a release-derived/open non-matching value as `bad_request` (should be empty/partial per OD-03), or accepting a static-closed violation (should be `bad_request`).
- **Drift-prone shims.** Reintroducing/retaining a hand-maintained materiality/condition table or prose string-splitting (O-1, O-8, G-4) creates silent behaviour change on contract rewording.
- **Nondeterminism.** Set/dict iteration order or model-driven slot values breaking repeatability and replay equality.
- **Conflated bad_request semantics.** Failing to separate the repairable user-identity `bad_request` (retry once, then honest refusal per ADR-014) from the planner-shape `bad_request` (defect) — the two require opposite dispositions.
- **Additive-scope creep.** Constructed requests carrying schema-forbidden fields (`additionalProperties: false` violation) would be rejected by the emitter and, if mishandled, surface as the very masked failure this work item exists to prevent.
- **Gated/excluded confusion.** Routing VC-POC-14 or excluded topics through the bad-request-failure path instead of the registry-owned `excluded_question`/refusal path, or vice versa.

*Boundary note: this analysis frames requirements, risks, and hypotheses only. It proposes no code changes and makes no determination of whether the current implementation passes; those remain host-owned decisions informed by a later current-state inspection.*
