# Implementation Plan

## Bounded objective
Make the Claude subscription-account transport (`harness/claude_account_cli.py`) populate the `RequestUsage` on the `ModelResponse` it returns so the existing telemetry (`harness/usage.py`, ADR-005) records the Claude CLI's actual token usage — aggregate input/output plus `cache_read`/`cache_write`, and the per-model breakdown from the CLI's `modelUsage` (including any helper-model entry) — while continuing to emit no raw CLI response body and no credential/token. Correct the stale RUNBOOK.md statement that the Claude CLI does not expose cache counters (concern C1).

Today `_invoke_claude` returns `ModelResponse(parts=[ToolCallPart(...)], model_name=…, provider_name="claude-account", finish_reason="tool_call")` with NO `usage=`, so every account-mode request is recorded as all-zero tokens and the ADR-006 cache-hit regression view can never fire on the local POC path. That is the whole gap.

## Definition Pack interpretation
Three pinned references, all in k-12Harness:
- RUNBOOK.md §1: says the SDK-only cache-counter live test is skipped "because Claude CLI does not expose those counters." Concern C1 records this as stale for claude 2.1.168; in scope to correct.
- docs/architecture.md §9: telemetry is one JSONL line per request + one per run, DuckDB at read time; constrains shape (one request line, token quartet, cache hits) but does not forbid account-mode usage.
- docs/decisions.md: ADR-005 (JSONL+DuckDB), ADR-006 (cache regression = cache_read_tokens>0), ADR-018/019 (account transport). ADR-018 explicitly asserts "Account plans expose neither per-request USD pricing nor provider cache counters through this CLI boundary," which contradicts the request and C1; docs/decisions.md is append-only, so it cannot be edited in place (see gaps).

Token convention is authoritative in usage.py and test_usage.py: pydantic-ai RequestUsage follows genai-prices where input_tokens INCLUDES cache read+write, whereas Anthropic/CLI reports input excluding cache (USAGE_SONNET: 1000 uncached + 2000 read + 100 write ⇒ input_tokens=3100). Any mapping added must obey this.

## Current implementation
- `harness/claude_account_cli.py::_invoke_claude` decodes stdout (via `_result_envelope`/`_structured_action`) and builds a `ModelResponse` with no usage. The success envelope carries a top-level `usage` block (test_session_limit_… shows `cache_creation_input_tokens`); C1 asserts it also carries `cache_read_input_tokens`, `input_tokens`, `output_tokens`, and a per-model `modelUsage` map.
- `harness/usage.py` already reads input/output/cache_read/cache_write and `details = dict(message.usage.details or {})` into `LlmRequestRecord`, and sums them into `RunSummaryRecord`. It needs NO change once the adapter supplies a populated `RequestUsage`. `LlmRequestRecord` has a single `model` column (SCHEMA_VERSION=1); per-model breakdown has no first-class column and would ride in `details`.
- `harness/agent.py::_finish` already calls `usage.record_run` on every branch — no agent change needed.
- Live tests encode the stale assumption: `test_the_frozen_prompt_is_actually_cached` skips for claude_account; `test_a_live_run_stays_inside_the_control_envelope` asserts `not complete` (cost unpriced — still true; tokens ≠ cost). pytest.ini excludes `-m live` from the default gate, and `tests/unit/test_live_tier.py::test_live_eval_file_is_deselected_by_default_and_fully_marked_live` (PF-4) enforces that every test in tests/live carries the `live` marker — so any new live test must be marked or the unit suite fails.

## Proposed changes
- **harness/claude_account_cli.py** — the substantive change.
  - Add a pure helper `_request_usage(envelope: dict) -> RequestUsage` that: maps `cache_read_input_tokens→cache_read_tokens`, `cache_creation_input_tokens→cache_write_tokens`, `output_tokens→output_tokens`, and sets `input_tokens = base_input + cache_read + cache_write` (genai-prices convention that usage.py/test_usage.py assume); reads the per-model `modelUsage` map (main + any helper model), aggregating its counts into the returned totals when present (so helper-model tokens enter the aggregate) and recording the per-model breakdown as integers-only, model-name-keyed entries in `RequestUsage.details`; tolerates a missing/partial `usage` block by returning zeroed usage rather than raising (ADR-005 telemetry-never-fails posture); emits ONLY numeric counts and model names — never the CLI `result` text, never any token/credential.
  - Refactor `_invoke_claude` to decode the success envelope once and pass `usage=_request_usage(envelope)` into the returned `ModelResponse`; no change to action extraction or the error/quota paths; `_redact_account_token` stays on every error string.
- **tests/unit/test_claude_account_cli.py** — the coverage/enforcement file for this adapter (every helper already has a dedicated test), so the new symbol must gain: a test that a `usage` block maps into RequestUsage with cache read/write set and input including cache; a test that `modelUsage` with a main AND a helper model produces an aggregate summing both and a `details` per-model breakdown; a test that no raw `result` text and no token appear in `usage`/`details`; a test that a missing/empty `usage` block yields zero usage without raising; and end-to-end that `build_claude_account_model` returns a ModelResponse with populated usage when the fake CLI returns a usage block.
- **harness/usage.py** — NO code change; it already surfaces the four token fields and `details`. tests/unit/test_usage.py is the enforcing invariant for the token convention the mapping must satisfy and MUST remain green unchanged (it is why input_tokens is summed rather than passed through).
- **RUNBOOK.md** — correct the §1 sentence claiming the CLI does not expose cache counters, to reflect that claude now returns `cache_read_input_tokens`/`cache_creation_input_tokens` and per-model `modelUsage` and that the harness records them (per C1).
- **releases/_next/<slug>.md** — the repo's release-notes CI gate (`.github/workflows/pr-checks.yml` job `release-notes` and `.claude/hooks/require-release-notes.sh`) requires exactly one new `releases/_next/*.md` with a `**Type:** patch|minor|major` line and `## Local deployment`, `## Manual steps`, `## Verification` sections (the last with a fenced ```bash block), and forbids touching VERSION or adding `releases/v*.md`. This change must add one such file (Type: patch) or the PR gate fails.

Named but deliberately out of core scope (not to be implemented around silently — see gaps): tests/live/test_live_eval.py (cache-counter skip + `not complete` assertion encode the same stale claim; reconciling is a live-tier behavioural decision, and any new live test must be `live`-marked per PF-4); docs/decisions.md ADR-018 cache-counter claim (append-only ADR log ⇒ a superseding ADR is an authority decision, not an edit).

## Verification approach
- Deterministic, offline: `.venv/bin/python -m pytest tests/unit/test_claude_account_cli.py -q` (new usage-parsing tests) and `.venv/bin/python -m pytest tests/unit/test_usage.py -q` (must stay green, proving the convention held).
- Full suite (manifest verify): `.venv/bin/python -m pytest tests/ -q`.
- Acceptance: a fake CLI success envelope carrying `usage` + `modelUsage` yields a ModelResponse whose usage has non-zero input/output/cache_read/cache_write; via `usage.extract_request_records` the LlmRequestRecord shows those values plus a per-model `details` breakdown, and neither the record nor `details` contains the CLI `result` text or the account token.
- Release-notes gate: mirror it locally — the added `releases/_next/*.md` must satisfy the four section/Type checks in `.claude/hooks/require-release-notes.sh`.
- Live tier (tests/live, -m live, manual, never a PR gate): not required to pass here; enabling the account-mode cache-read assertion is gated on the helper-model/cache-read gaps below.

## Risks, assumptions and gaps
- Assumption: the CLI top-level `usage` block uses Anthropic-style names (`input_tokens`, `output_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`) reporting input EXCLUDING cache (C1). Only `cache_creation_input_tokens` is confirmed by an existing repo fixture; the rest are asserted by C1, not pinned.
- Assumption: `modelUsage` is a map keyed by model name with per-model counts; the helper-model entry shape is unobserved (C1).
- Risk: if `modelUsage` totals and the top-level `usage` block disagree, the plan uses the `modelUsage` sum when present, else the top-level block, and records both in `details` for auditability.
- Risk: recording per-model only in `details` may not satisfy a stricter reading of "per-model"; a first-class column would be a SCHEMA_VERSION bump (larger scope).
- ADR-005's telemetry-never-fails posture is preserved by the total-tolerant parser and by record_run already swallowing errors.

## Planning gaps
GAP: request.md ("record … cache and helper-model usage") vs docs/decisions.md ADR-018 ("Account plans expose neither per-request USD pricing nor provider cache counters through this CLI boundary") — ADR asserts counters unavailable; append-only ADR log needs a superseding decision to authorise recording them.
GAP: request.md ("helper-model usage") vs manifest concern C1 ("Helper-model entries in modelUsage not yet observed") — no captured sample of a helper-model modelUsage entry, so its keys/shape are unverified.
GAP: exact CLI usage/modelUsage field names — only usage.cache_creation_input_tokens is pinned by a repo fixture (tests/unit/test_claude_account_cli.py); cache_read_input_tokens, input_tokens, output_tokens, and the modelUsage structure are asserted only by concern C1 and not captured under work-store evidence.
GAP: per-model granularity — request.md implies per-model recording but harness/usage.py LlmRequestRecord (SCHEMA_VERSION=1) has a single model column; unresolved whether details-dict capture satisfies the request or a schema_version bump is required (ADR-005).
GAP: RUNBOOK.md:55 skip rationale vs tests/live/test_live_eval.py (test_the_frozen_prompt_is_actually_cached skip and test_a_live_run_stays_inside_the_control_envelope `not complete` assertion) — both encode "account mode does not expose cache counters"; unresolved whether the live cache-read assertion should now be enabled for account mode.

## Size estimate
Small: the substantive change is one helper plus a two-line refactor in harness/claude_account_cli.py, its enforcing unit tests in tests/unit/test_claude_account_cli.py, one corrected sentence in RUNBOOK.md, and one mandated releases/_next note; harness/usage.py and the DuckDB views are unchanged and merely inherit the populated usage. It grows toward medium only if authority requires the ADR-018 superseding decision, the live-tier reconciliation, or a per-model schema bump — each deferred as a gap rather than folded in.

```yaml
size_estimate:
  files: 4
  scale: small
  subsystems: [harness, tests, docs]
```
