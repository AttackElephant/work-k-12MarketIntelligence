<!-- Derived view of 001-contract.yaml. The YAML is authoritative. Regenerate; never edit. sha256=01b8ac82c1a5861bdaa8ac392ece38dc2472adfbdfd0cd19eaba83c17a1eea3e -->

# Acceptance contract — work item 20260908T113226

contract_version: 0.1.0  ·  artifact_version: 1

## Observable behaviour
- **account-usage-recorded** — A claude_account request whose CLI success envelope carries a `usage` block returns a `ModelResponse` with a populated `RequestUsage`, so `harness/usage.py` records non-zero input/output/cache_read/cache_write instead of the current all-zero record.
- **per-model-recorded** — When the envelope's `modelUsage` map names a main model and any helper model, their counts are summed into the returned aggregate and the per-model breakdown is recorded as integer-only, model-name-keyed entries in `RequestUsage.details`.
- **missing-usage-safe** — A missing or partial `usage` block yields a zeroed `RequestUsage` and the request still records, rather than raising.
- **no-raw-no-secret** — The recorded usage and its `details` contain only numeric counts and model names — never the CLI `result` text, never any token or credential.
- **runbook-corrected** — RUNBOOK.md §1 no longer states that the Claude CLI does not expose cache counters; it reflects that claude returns `cache_read_input_tokens`/`cache_creation_input_tokens` and per-model `modelUsage`, which the harness now records (concern C1).

## Required outputs
- `harness/claude_account_cli.py` — Adds the pure `_request_usage(envelope)` helper and refactors `_invoke_claude` to pass `usage=` into the returned ModelResponse; the substantive change.
- `tests/unit/test_claude_account_cli.py` — Per-helper coverage/enforcement file for this adapter; must gain tests for usage mapping, helper-model aggregation, no-raw/no-secret, missing-usage tolerance, and end-to-end populated usage.
- `RUNBOOK.md` — Correct the stale §1 sentence claiming the CLI does not expose cache counters (concern C1).
- `releases/_next/claude-account-usage.md` — The release-notes CI gate requires exactly one new releases/_next/*.md (Type: patch) with Local deployment / Manual steps / Verification sections; the PR gate fails without it.

## Deterministic criteria
- **full-test-suite** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/ -q`
  - expected: The declared verification suite exits zero with no failures and no errors (pre-existing xfail/xpass unaffected).
  - machine-checked: yes
- **adapter-usage-tests** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_claude_account_cli.py -q`
  - expected: The adapter tests, including the new usage-parsing tests, exit zero with no failures.
  - machine-checked: yes
- **usage-convention-green** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_usage.py -q`
  - expected: The token-convention invariant (input_tokens includes cache read+write) stays green, proving the mapping obeyed it.
  - machine-checked: yes
- **release-note-typed** in `k-12Harness`
  - command: `grep -RE '^\*\*Type:\*\* (patch|minor|major)' releases/_next/`
  - expected: The added release note under releases/_next/ declares a patch/minor/major Type line, as the release-notes gate requires; the base commit has no such file so this depends on the change.
  - machine-checked: yes

## Must not change
- The genai-prices token convention enforced by harness/usage.py and tests/unit/test_usage.py: input_tokens INCLUDES cache read+write (USAGE_SONNET 1000+2000+100 => 3100). The mapping must satisfy it; neither file is edited.
- harness/usage.py contains no code change — it already surfaces the four token fields and details and must merely inherit the populated RequestUsage.
- LlmRequestRecord SCHEMA_VERSION (=1) and its single `model` column, absent the per-model schema-bump option in decision-per-model-granularity.
- The account-mode error and quota paths and `_redact_account_token` on every error string in harness/claude_account_cli.py; only usage population is added.
- The contract pin (contract/PINNED.json) and tests/unit/test_contract_pin.py; VERSION is not touched and no releases/v*.md is added (release-notes gate).

## Failure behaviour
- The CLI success envelope has a missing or partial `usage` block. → `_request_usage` returns zeroed usage and the request records normally, never raising (ADR-005 telemetry-never-fails posture).
- The `modelUsage` per-model sum disagrees with the top-level `usage` block. → The parser uses the `modelUsage` sum when present, otherwise the top-level block, and records both in `details` for auditability.

## Compatibility
- ADR-005: record_run/telemetry never fails a successful run; the total-tolerant parser preserves this.
- ADR-019: the account transport records account-plan usage only and leaks no credential or token; ADR-018 remains history and is superseded by ADR-019 for the local POC provider.
- ADR-006: the cache-hit regression check remains `cache_read_tokens > 0` in the usage telemetry, now populatable on the account path.
- ADR-007: the live tier (tests/live, -m live) stays manual and never becomes a PR gate; pytest.ini deselects -m live by default.
- The release-notes CI gate (.github/workflows/pr-checks.yml job release-notes and .claude/hooks/require-release-notes.sh): exactly one new releases/_next/*.md with a Type line and Local deployment / Manual steps / Verification (fenced bash) sections; VERSION untouched; no releases/v*.md added.

## Compatibility baselines
_none_

## Contract requirements
- **scope-is-the-file-list** — The permitted change set is exactly required_outputs plus permitted_files; harness/usage.py, harness/agent.py, tests/unit/test_usage.py, docs/decisions.md and tests/live are outside scope.
- **decision-per-model-granularity** — DECISION: how per-model (helper-model) usage is recorded. Options: (a) integer-only, model-name-keyed entries in RequestUsage.details with no schema change (the option the plan takes and this contract is written on); (b) a first-class per-model column via an LlmRequestRecord SCHEMA_VERSION bump. Affects: required_outputs (option (b) adds harness/usage.py and its migration and enforcing tests), must_not_change (the SCHEMA_VERSION=1 invariant), semantic per-model-details, observable_behaviour per-model-recorded.
- **assumption-cli-field-names** — ASSUMPTION: the CLI top-level `usage` block uses Anthropic-style names (input_tokens, output_tokens, cache_read_input_tokens, cache_creation_input_tokens) reporting input EXCLUDING cache (C1). Only cache_creation_input_tokens is confirmed by an existing repo fixture; the rest are asserted by concern C1, not pinned under work-store evidence.
- **gap-helper-model-shape** — GAP: the helper-model `modelUsage` entry shape is unobserved (concern C1). The parser tolerates and aggregates helper entries when present and records them in details; no captured helper sample pins their keys.
- **adr018-conflict-resolved** — ASSUMPTION: request.md vs ADR-018 ('Account plans expose neither per-request USD pricing nor provider cache counters through this CLI boundary') is resolved — ADR-018 (Codex boundary) is superseded by ADR-019 (T1 review), so recording cache/per-model counters needs no new superseding ADR; adding one is out of scope.

## Semantic criteria
- **usage-populated-per-convention** — A fake CLI success envelope carrying `usage` yields a ModelResponse whose RequestUsage sets cache_read_tokens, cache_write_tokens and output_tokens from the CLI counts and input_tokens = base_input + cache_read + cache_write; via usage.extract_request_records the LlmRequestRecord shows those non-zero values.
- **helper-model-aggregated** — An envelope whose `modelUsage` names a main and a helper model produces an aggregate summing both, and a per-model breakdown in RequestUsage.details keyed by model name.
- **per-model-details** — Per-model usage is recorded as integer-only, model-name-keyed details entries (option (a) of decision-per-model-granularity); no first-class column is added.
- **no-raw-response-no-credential** — Neither the RequestUsage, its details, nor the resulting LlmRequestRecord contains the CLI `result` text, any raw response body, or any token/credential — only numeric counts and model names.
- **telemetry-never-fails** — A missing/empty/partial usage block produces zeroed usage and a recorded request without raising.
- **runbook-reflects-c1** — RUNBOOK.md §1 no longer claims the Claude CLI does not expose cache counters and states the harness now records cache and per-model usage (C1).

## Out of scope
- Reconciling tests/live/test_live_eval.py (the cache-counter skip and the `not complete` assertion) — a live-tier behavioural decision, and any new live test must carry the `live` marker (PF-4).
- Enabling the account-mode cache-read assertion in the live tier; deferred pending captured helper-model/cache-read evidence.
- Adding a superseding ADR to docs/decisions.md for the ADR-018 cache-counter claim (append-only log; resolved as superseded by ADR-019).
- A first-class per-model column or LlmRequestRecord SCHEMA_VERSION bump (see decision-per-model-granularity), and any change to harness/usage.py or harness/agent.py.

## Base-commit dry run

Every deterministic criterion run once at `k-12Harness` @ 1b3284534ce9, before any change (2026-09-27T12:50:19Z). A flag means the result cannot depend on the change.

- **full-test-suite** — ran, exit 0 on the base
- **adapter-usage-tests** — ran, exit 0 on the base
- **usage-convention-green** — ran, exit 0 on the base
- **release-note-typed** — ran, exit 1 on the base

_no criterion flagged_
