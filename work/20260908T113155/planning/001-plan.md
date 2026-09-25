# Implementation Plan

## Bounded objective

Make `evals/live.py` (k-12Harness / execution repo) actually **execute and grade three genuinely isolated stages** selected by the already-present `--stage {intent,narration,end-to-end}` flag, instead of running the full end-to-end pipeline regardless of the flag. Concretely:

- **intent** stage: run only Stage 1 (`harness.intent.interpret_intent`, NL → `StructuredIntent`: question id / excluded / unknown, school mentions, filters, missing material inputs), graded by the decision-scoped graders only; no evidence fetch, no rendering.
- **narration** stage: run only Stage 4 (`harness.narration.narrate` over an `AnswerView` built by `harness.answer_view.build_answer_view` from a fixed/oracle envelope per case — NOT the model's own fetched envelope), graded by the rendering/grounding/limitations graders only, so narration quality is measured independently of intent quality.
- **end-to-end** (and `--stage` omitted): the existing full trial — unchanged.

All three must preserve the runner's current operational guarantees, which already exist and must not regress: quota-aware interruption (`QuotaExceeded` → `SuiteInterruption(kind="quota_exhausted", reset_at=...)` → `break`), `CredentialsRejected` stop-with-no-report, checkpoint-after-every-trial with atomic `.tmp`+`.replace` writes, `--resume` that skips completed trials via the `_trial_key`/`done` set, and completed-trial-only aggregation of pass@k / pass^k (`completed_reports`/`aggregate`).

Out of scope: the evidence contract (v0.3.0 pinned), production agent decision logic, the deterministic stub-tier pass/fail rules, and the golden-case authoring format beyond any per-stage oracle wiring narration requires.

## Definition Pack interpretation

The manifest pins the v0.3.0 contract set (`contract/PINNED.json`, registry, schemas), `docs/decisions.md`, `evals/cases/golden_cases.json`, `prompts/input_catalog.snapshot.json`, `prompts/rules.md`.

- The **contract files** govern what a correct answer/refusal *is* (limitations rendered verbatim, no harness-assigned grade — ADR-009; refusal text sourced from the registry exclusion lists; error taxonomy). These constrain narration and end-to-end grading and must not change.
- **`docs/decisions.md` is the only pinned source that speaks to the live tier.** ADR-007: two tiers; the live tier reports pass@k (capability) and pass^k (reliability) separately, is manual/nightly, never a PR gate. ADR-013: grading reads the transcript / the envelope the tool actually returned, never the model's account of its own work — the authority basis for grading isolated narration against an oracle envelope. ADR-014: the pipeline stages have one owner each; do not duplicate decision or render logic in the eval runner. ADR-015 consequences: `python -m evals.live` is runnable/manual and reports the two numbers separately. ADR-016: preserve credential/transport handling (status-based classification via `classify`; `CredentialsRejected` stops with no report; 500/timeout is an ordinary failed trial; `cost_complete` requires that something was priced). ADR-018→ADR-019: the live tier consumes **account subscription quota** (`claude_account`, superseding `codex_account`); account plans expose no per-request USD, so `cost_complete` is expected False there (asserted in `tests/live/test_live_eval.py`).
- **The terms the request turns on — “genuinely isolated” stage semantics, oracle-narration input, per-stage grader sets — have no schema/ADR that specifies them exactly.** They are interpreted from the request plus ADR-013's isolation principle and the existing four-stage pipeline (`_ask_pipeline` in `harness/agent.py`). Recorded as a gap.

## Current implementation

Confirmed from the provided source of `evals/live.py`, `evals/graders.py`, `evals/schema.py`, `evals/replay.py`, `harness/intent.py`, `harness/narration.py`, `harness/answer_view.py`, `harness/agent.py`, and the two test modules:

- **`--stage {intent,narration,end-to-end}` already exists** (`build_parser`), default `None`. It is validated (`intent`+`pressure` rejected; `end-to-end` requires k=1), written into the checkpoint (`payload_for` and `selection`), and guarded on resume (a resume whose recorded `stage` differs is rejected). `tests/unit/test_live_tier.py` covers exactly this parsing/validation/checkpoint/resume behaviour.
- **But `stage` never reaches execution or grading.** `run_suite` and `run_trial` take no `stage` argument; `run_trial` always calls `ask(agent, deps, phrasing)` (the legacy full loop) and grades with `replay_mod.project(...)` → `grade_case` → `GRADERS_BY_DIRECTION[direction]` (the full grader set). So today `--stage intent` and `--stage narration` run and score an identical end-to-end trial to the default; the flag only changes report metadata and the k/pressure validation. This is the exact gap the request closes.
- **The isolated stages exist and are individually invokable.** `harness.intent.interpret_intent(question, model=None)` runs one no-tool model call → `StructuredIntent`. `harness.answer_view.build_answer_view(envelope)` is a pure function → `AnswerView`. `harness.narration.narrate(view, deps, model=None)` runs one no-tool call → `NarrationOutput` and validates via `rendering.validate_answer` against `deps.last_envelope`. `harness.agent._ask_pipeline` already chains all four; the eval runner can call the isolated entry points directly rather than the whole pipeline. All accept a `model=` injection point for scripting (mirroring `evals/replay.scripted_model`).
- **Grader partition is available.** `evals/graders.py` graders are transcript-based (`RunRecord`) and already split by direction. There is a clean decision-vs-rendering split: decision graders (`question_match`, `request_match`, `school_resolved_through_directory`, `clarification_correct`, `refusal_is_contract_sourced`) vs rendering graders (`output_branch`, `envelope_preservation`, `limitations_verbatim`, `confidence_unchanged`, `grounding`, `no_recommendation_language`, `no_forecast_framing`, `empty_stated`, `gaps_stated`, `injection_resisted`).
- **`RunRecord` shape constraint.** `evals/replay.project` builds `RunRecord` from `deps.tool_calls` and the union output. An intent-only run produces no rendered output and (for narration) no fetch call, so `project`/`RunRecord` cannot represent these shapes without either injecting a synthetic envelope tool-call or adding a stage discriminator — relevant to grader attribution (`last_envelope()`, `evidence_calls()`).
- Preservation machinery already present and correct: `QuotaExceeded` → `SuiteInterruption`; `CredentialsRejected` from `run_trial` via `classify`→`AuthError`, caught in `main` to exit without a report; ordinary exceptions → failed `TrialResult`; `on_progress=checkpoint` after each trial; atomic `write_report`; `_trial_key=(case_id, phrasing, trial)` + `done` set drive resume; `completed_reports`/`aggregate` count only fully-completed cases; `max_trials` chunk limit.

## Proposed changes

- **`evals/live.py`** — primary. Thread `stage` into `run_suite` → `run_trial` (already passed through `selection`/`slots`; add the parameter and pass it from `main`). Branch execution inside `run_trial`:
  - `intent`: call `interpret_intent(phrasing, model=...)`; build a stage-tagged record of the intent decision; grade with an intent-scoped grader set (question/request/clarify/refuse). No `EvidenceClient` fetch.
  - `narration`: load the case's canonical/oracle envelope (the fixture selected by `case.stub_case`, obtained by running the stub emitter once or reading the fixture), build the `AnswerView` via `build_answer_view`, set `deps.last_envelope`, call `narrate(view, deps, model=...)`; grade with a narration-scoped grader set against that oracle envelope so a wrong intent cannot affect the score.
  - `end-to-end` and `None`: keep the current `ask(build_agent(), deps, phrasing)` path and full grader set unchanged.
  Keep `CredentialsRejected`, status-based `classify`, ordinary-failed-trial, quota interruption, `cost_complete`, checkpoint/atomic-write, and resume semantics identical. `spend_usd(deps.last_messages)` still applies where a model call occurred.
- **`evals/live.py` (accounting)** — confirm `summarise`/`aggregate`/`completed_reports` count each completed stage-scoped trial exactly once. No `_trial_key` change is needed because `stage` is fixed per invocation and resume already forbids cross-stage mixing; the checkpoint `stage` guard remains the sole cross-stage safety.
- **`evals/graders.py`** — add stage-scoped grader selection (`INTENT_GRADERS`, `NARRATION_GRADERS` derived from the existing partition) and a `grade_case(case, record, stage=None)` dispatch; `stage=None` preserves `GRADERS_BY_DIRECTION` exactly. Do not alter any existing grader's pass/fail rule.
- **`evals/schema.py` / `evals/replay.py`** — add a minimal stage discriminator (and/or an oracle-envelope tool-call injection) to `RunRecord`/`project` only if needed so intent-only (no rendered output) and narration-only (oracle envelope, no fetch) runs are attributed correctly and `last_envelope()`/`evidence_calls()` do not misfire. Prefer the smallest change that lets existing graders read the oracle envelope for narration.
- **`harness/intent.py` / `harness/narration.py` / `harness/answer_view.py`** — no logic change expected; reuse the existing public entry points (ADR-014 one-owner rule). Add a thin eval-facing helper only if a stage cannot be invoked in isolation from the current signatures.
- **`tests/unit/test_live_tier.py`** — extend deterministic offline tests using a scripted model: (a) narration grading uses the oracle envelope and is unchanged by a deliberately wrong scripted intent (proves isolation); (b) intent grading ignores prose/rendering graders; (c) checkpoint→resume skips completed trials per stage; (d) pass@k/pass^k over completed trials only, per stage; (e) quota interruption leaves a resumable checkpoint; (f) `CredentialsRejected` writes no report; extend `test_chunked_suite_resumes_without_repeating_completed_trials` per stage.
- **`tests/live/test_live_eval.py`** — add manual/credentialed coverage of the three-stage surface; remains `live`-marked and excluded from the PR gate.

## Verification approach

- Manifest gate: `.venv/bin/python -m pytest tests/ -q`. Load-bearing deterministic evidence in `tests/unit/test_live_tier.py`, run with a scripted model (mirroring `evals/replay.scripted_model`, as existing tests do):
  - **Isolation (the defining test):** a narration-stage trial fed a wrong scripted intent still passes/fails purely on the oracle envelope's rendering — grounding, `limitations_verbatim`, `confidence_unchanged`, `no_forecast_framing` all read the oracle envelope, not the intent.
  - Intent-stage trial grades only question/request/clarify/refuse; rendering graders do not run.
  - Resume after a `max_trials` chunk re-runs no completed trial and yields identical per-stage pass@k/pass^k.
  - `test_quota_stops_without_recording_a_model_failure` and `test_a_rejected_credential_*` still hold under each stage.
  - Completed-trial accounting: `summarise`/`aggregate` match a hand-computed trial matrix per stage; `cost_complete` gating unchanged.
- Acceptance cases (`evals/cases/golden_cases.json`): `should_answer` across all three stages; `should_clarify` at intent (correct missing material input, no evidence fetched first); `should_refuse` (registry-sourced text preserved) at intent and end-to-end; injection cases (`canary_absent`, `canary_absent_and_limitations_verbatim`, `behavioural`) at narration and end-to-end.
- Regression: entire offline suite stays green; `tests/unit/test_contract_pin.py` untouched. Live tier stays manual per ADR-007.

## Risks, assumptions and gaps

- **Confirmed by the owner (concern C1):** “isolated narration” means grading against a canonical/oracle envelope per case (the existing `stub_case` fixture) built into an `AnswerView`, not the model's own fetched envelope and not a live fetch. Acquisition is fixed in “Planning review 1 resolutions” below.
- **Assumption:** default (`--stage` omitted) and `end-to-end` both keep the current full-loop execution; their only difference stays the k constraint already enforced.
- **Risk:** intent-only / narration-only runs do not fit the current `RunRecord`/`project` shape (an intent run has no rendered output; a narration run has no fetch call), forcing a small `schema.py`/`replay.py` change and risking `envelope_preservation`/`grounding` misgrading if the oracle envelope is not attributed by stage.
- **Risk:** k-12Harness is `diff-allowlist` governance with `scope.declared_files: []` / `declared: false`; a change spanning `evals/live.py`, `evals/graders.py`, possibly `evals/schema.py`/`evals/replay.py`, and two test files may exceed an unstated allowlist.
- **Risk:** quota-exhaustion signalling for the active `claude_account` transport (distinct from `CredentialsRejected` and from an ordinary 500) is assumed to already surface as `QuotaExceeded` via `classify`; the exact raise site is unconfirmed from available source.
- **Assumption:** narration-stage runs still record spend via `deps.last_messages`; under `claude_account` `cost_complete` remains False by design.

## Planning gaps

GAP: request.md requires executing/grading “genuinely isolated” stages, but no Definition Pack reference (contract v0.3.0 files, docs/decisions.md, golden_cases.json, input_catalog.snapshot.json, rules.md) defines stage-isolation or oracle-narration input semantics — request term lacks an authority source; interpreted via ADR-013's transcript/envelope principle.
GAP: `evals/live.py` already ships an inert `--stage` surface (validated, checkpointed, resume-guarded) whose value never reaches `run_suite`/`run_trial`/grading, versus request.md which assumes the stages are executed — conflict between current source behaviour and the request's premise; the work is “wire the existing flag into execution and grading”, not “add a flag”.
GAP: request.md requires “quota-aware checkpointing, resumability, completed-trial semantics” with no defining source; docs/decisions.md ADR-016 defines only credential/transport-failure handling, leaving quota-exhaustion signalling and checkpoint format undefined — conflict: request.md vs docs/decisions.md coverage.
GAP: account-quota provider authority is ambiguous — docs/decisions.md ADR-018 (`codex_account`) versus ADR-019 (`claude_account`, superseding) both present in the pack; the live tier's governing quota transport is asserted but not reconciled in the manifest.
GAP: k-12Harness scope governance is `diff-allowlist` with manifest `scope.declared_files: []` / `declared: false`; a multi-file change (evals/live.py, graders.py, possibly schema.py/replay.py, tests) may exceed the unstated allowlist — conflict: request.md multi-file scope vs manifest scope declaration.
GAP: `evals/schema.RunRecord`/`evals/replay.project` cannot represent an intent-only run (no rendered output) or a narration-only run (oracle envelope, no fetch call) without a stage discriminator or synthetic envelope injection, so `last_envelope()`/`evidence_calls()`-based graders may misattribute — conflict: request.md three-stage grading vs current RunRecord/project construction.
GAP: golden_cases.json encodes single end-to-end expectations per case; whether narration needs per-case oracle envelopes distinct from existing stub fixtures, and whether intent needs distinct expected fields, is unspecified — conflict: request.md three-stage grading vs golden_cases.json/current fixtures.

## Planning review 1 resolutions

These resolve PF-1..PF-4 of `reviews/001-planning-review.json`. Where they are more specific than the sections above, they govern.

**PF-1 — oracle envelope: confirmed, and acquisition is fixed.** The owner confirmed the oracle-envelope interpretation (concern C1). The narration oracle is the envelope the stub emitter returns for `STUB_EMITTER_CASE=case.stub_case`, obtained through `build_client("stub", case.stub_case)` with the case's recorded request — the same offline, deterministic mechanism the golden cases were generated with. It is acquired once per case before the trial's model path runs. It is never the real emitter and never a model-driven fetch. Consequences:
- `--stage narration --emitter real` is refused at argument validation (the stage is defined over the oracle, not the data project).
- A case with no `stub_case` is not narration-eligible: it is excluded from a narration selection and the excluded count is printed. A narration run whose selection has no eligible case exits non-zero rather than reporting an empty success.
- “Reading the fixture directly” is not used.

**PF-2 — credential rejection and checkpoints.** “No report is written” means `main` writes no final report after a rejected credential; it does not delete a checkpoint that already exists. Expected artifact state:
- Rejection before any trial completes: no report file and no transcripts file exist.
- Rejection after one or more completed trials: the last checkpoint (status `running`) is left byte-for-byte unchanged and remains resumable with `--resume`; no final report is written over it.
- The exit message is made truthful: it says whether a checkpoint exists and, if so, prints its path and the resume command, instead of always saying “No report was written”.
`tests/unit/test_live_tier.py` covers both cases for every stage.

**PF-3 — the isolation test is falsifiable.** The “wrong scripted intent” check is implemented as a perturbation plus direct assertions, not by feeding an intent into narration:
- Two narration-stage runs of the same case use scripted models that differ only in what intent they would produce if intent interpretation were invoked (one correct, one deliberately wrong). The `AnswerView` passed to `narrate`, the narration output and every grade are identical across the two runs.
- In a narration-stage trial, `interpret_intent` is never called and the agent's evidence tool is never invoked (spies); the only envelope in the record is the oracle acquired before the trial, and the narration graders read that envelope.
- The same spies assert the converse for the intent stage: no evidence fetch, no `narrate` call.

**PF-4 — live tests provably stay out of the default gate.** `live-eval-collect-only` only proves collection. `tests/unit/test_live_tier.py` gains a test that runs inside the contract's `full-test-suite` gate and asserts:
- collecting `tests/live/test_live_eval.py` with the repository's default options selects zero tests (every test there is deselected by `addopts = -m "not live"`);
- collecting it with `-m live` selects every test in the file, so no test lacks the `live` marker.
A missing marker therefore fails the PR gate.

No file is added to the scope by these resolutions: all changes fall in `evals/live.py` and `tests/unit/test_live_tier.py`, which are already listed.

## Size estimate

The core change is in `evals/live.py` (thread `stage` into `run_suite`/`run_trial` and branch execution for intent-only via `interpret_intent` and narration-only via `build_answer_view`+`narrate` over an oracle envelope, reusing existing harness stage logic), plus stage-scoped grader selection in `evals/graders.py`, deterministic tests in `tests/unit/test_live_tier.py`, and manual coverage in `tests/live/test_live_eval.py`; a small change to `evals/schema.py`/`evals/replay.py` is contingent on whether `RunRecord`/`project` can represent intent-only and narration-only runs without a stage discriminator. No contract or production-loop change. Medium: bounded to the eval subsystem and its tests, but spanning coupled files and touching accounting/resumption invariants that must be proven deterministically. If schema changes are avoidable it is ~4 files; otherwise 5.

```yaml
size_estimate:
  files: 4
  scale: medium          # small | medium | large
  subsystems: [evals, tests]
```
