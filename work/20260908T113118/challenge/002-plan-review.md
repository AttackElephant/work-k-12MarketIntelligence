# Independent Plan Review

This review reads the returned code (`harness/planner.py`, `harness/contract.py`, `harness/intent.py`, `harness/errors.py`, `config/settings.py`, `tests/unit/test_planner.py`, `tests/unit/test_agent_loop.py`), the plan (`planning/001-plan.md`), and the current contract (`acceptance/002-contract.yaml`) against the Definition Pack and blind analysis. I make no pass/fail, materiality, or reconciliation-required determination.

One authority question the blind analysis left open is settled by the returned code: `harness/contract.material_inputs_for()` and `conditional_material_for()` read the pinned v0.3 `input_policy` directly, and the docstring states "nothing here parses prose for a 'when …' condition any more." So the `settings.MATERIAL_INPUTS` shim (blind Q1) and the G-4 string-splitting hazard are already gone. That narrows the review to the plan's additions.

## Approach challenges
1. **"Every planner bad_request is a harness defect" is contradicted by the plan's own direct-ID passthrough.** The plan raises `TerminalError` on `bad_request` (`error-boundary-refinement`), yet `POC_CONTRACT_V0.3_SPEC.md` classifies an absent-directory `acara_sml_id` as `bad_request`, and the plan deliberately leaves caller-supplied numeric IDs unvalidated (`out_of_scope`). A user typo therefore reaches the emitter and becomes an operational crash. Disproof: no path lets an unvalidated numeric ID reach `fetch_evidence` — but the plan forbids adding that validation, so the tension is demonstrated, not missing evidence. (C1)
2. **The planner still repairs.** It retains headcount/non-teaching scope cleanup and adds normalization; if normalization is not total over the closed vocabularies, an emitter reject becomes a crash. Disproof: a test showing every non-canonical selector normalizes or fails before dispatch.
3. **"Exactly one directory record per mention" can regress answerable questions into clarifications** because directory search is substring/fuzzy (`poc_school_directory` `school_search`). Current code takes `resolved[0]`. Disproof: golden answer cases route only through the agent loop, or production returns exactly one record. Missing evidence: no live directory sample (ADR-016).

## Requirement coverage gaps
1. **VC-POC-09/10 "at least one explicit filter beyond the sector default" (OD-07) unaddressed.** Deriving from `input_policy.required` (`peer_filters` present) accepts an only-defaults request; the emitter rejects it as `bad_request` → now `TerminalError`, versus the golden clarify expectation. Disproof: a test asserting clarification for only-defaults.
2. **No whitelisting of `intent.filters`.** It is free-form `dict[str, Any]`; a stray key hits `additionalProperties:false` → `bad_request` → `TerminalError`. Only three request schemas are supplied (C2).
3. **Silent drop of `requested_metrics` for no-selector questions** (e.g. VC-POC-05) is unstated.
4. **teaching staff→teaching_fte alias co-owned** by `rules.md` (must_not_change) and the planner, with no cross-check (blind Q5).

## Unsupported architecture assumptions
1. **Runtime input-catalog load for static vocabularies.** `profile_metrics`/`student_context_metrics`/`ratios` are `static_closed` and available from the pinned registry/snapshot; a runtime emitter call adds data-project dependency and non-determinism (ADR-016). Disproof: planner reads static vocabularies from `contract`/snapshot.
2. **Assumes the client serves the catalog** — only `school_directory`/`fetch_evidence` evidenced; `get_input_catalog` is a tool. Missing evidence (`evidence.py`/`tools.py` not returned).
3. **Assumes the planner is production and the golden baseline exercises it** — `test_agent_loop.py` replays golden cases through the legacy loop; `run.py` wiring not returned. Missing evidence.
4. **"Deterministic from intent" contradicted** by `request_scope_problem(deps.user_question, …)`, which the plan preserves.

## Contract testability and achievability
1. **All deterministic gates are exit-code-only with `-k` filters that can pass vacuously** (no minimum-count or named-test assertion). Disproof: add expected counts/named assertions.
2. **"Fail closed"/"fail before dispatch" does not name the typed outcome** (`identity-conflict`, normalization failure) — ambiguous between TerminalError/Clarify/Unsupported.
3. **`bad-request-failure` is universally quantified but true only under the stub**; the 99999 empty-success case diverges on the real emitter (C1).
4. **Freezing the O-8 xfail is an implicit, unstated requirement.** The plan implements conditional distance in the planner but not in the agent-loop `_validate`; the strict xfail in `test_agent_loop.py` must remain xfailing for `full-test-suite` to exit 0, creating an unsurfaced two-path divergence.
5. **VC-POC-03→`ratios` routing authority unnamed** (`metric_list` is null, no `requested_ratios`); a hard-coded map risks violating `pinned-policy-authority`.
6. **"catalog-backed allowed values where available" is soft** — always emitting `[]` conforms.

## Blind findings disposition
- Q1: resolved (contract.py reads pinned policy).
- Q2: resolved for construction defect; user-bad-ID sub-case unanswered.
- Q3: partially resolved (planner path fixes it; agent path unchanged; divergence/freeze unsurfaced).
- Q4: resolved (`no-invented-defaults`; `defaultable_keys()`).
- Q5: unanswered (dual ownership, no drift test).
- Q6: partially resolved (zero/multi→clarify; missing-geography→unanswerable; absent-ID now crashes).
- Q7: largely resolved (fixed-intent tests; but `deps.user_question` reliance; no byte-identical assertion required).
- Q8: resolved for planner logic (reads only required directory field); live directory untested.

## Falsifiable questions for reconciliation
1. For a user-supplied numeric ID the emitter rejects, is the intent an operational TerminalError crash or a graceful outcome — and what distinguishes a constructed-malformed request from a passed-through user error?
2. Is normalization provably total over the pinned closed vocabularies so a normalized request cannot itself trigger emitter bad_request?
3. Does the planner enforce VC-POC-09/10 "at least one filter beyond the sector default," or dispatch and crash?
4. Are `intent.filters` keys whitelisted against the supplied request schemas before dispatch?
5. Which pinned artifact is authoritative for VC-POC-03→ratios routing?
6. Is `plan_and_execute` on the live path, and do the cited golden baselines exercise it or only the legacy loop?
7. Does normalization read `deps.user_question`, and how is "deterministic from intent" preserved and tested?
8. Does the planner call the live emitter `--input-catalog` at runtime, or read pinned static vocabularies?
9. For "cannot be normalized" and "identity conflict," what exact typed outcome is required?
10. Is leaving the agent-loop O-8 bug in place (strict xfail) a deliberate, stated requirement while the planner path fixes conditional materiality?
