Only the declared editable files changed.

## Summary

**Files changed** (all within the declared editable set):

- **`harness/claude_account_cli.py`** — added `_usage_block_counts` and `_request_usage(envelope)` pure helpers, and wired `_invoke_claude` to decode the success envelope's `usage`/`modelUsage` and pass `usage=_request_usage(envelope)` into the returned `ModelResponse`. Aggregates main + helper-model `modelUsage` counts into the returned totals (falling back to the top-level `usage` block when `modelUsage` is absent), sets `input_tokens = base_input + cache_read + cache_write` per the genai-prices convention, records an integer-only, model-name-keyed per-model breakdown plus the raw top-level totals in `details` for auditability, and tolerates missing/partial/malformed blocks by returning zeroed usage rather than raising. No change to the error/quota paths or `_redact_account_token`.
- **`tests/unit/test_claude_account_cli.py`** — added tests for: usage-block mapping per the cache-inclusive convention; helper-model aggregation into the total plus per-model `details`; absence of raw `result` text/credential in usage/details; tolerance of missing and partial `usage` blocks; tolerance of a malformed `modelUsage` value; and an end-to-end `_invoke_claude` test (fake subprocess) proving `build_claude_account_model`'s returned `ModelResponse` carries populated usage.
- **`RUNBOOK.md`** — corrected §1's stale claim that "Claude CLI does not expose those counters," describing instead that the CLI returns `cache_read_input_tokens`/`cache_creation_input_tokens` and per-model `modelUsage`, now recorded by the harness (the account-mode live cache-read assertion itself stays deferred, per contract's out-of-scope item).
- **`releases/_next/claude-account-usage.md`** — new release note (`Type: patch`) with Summary/Changes/Local deployment/Manual steps/Verification (fenced bash)/Rollback sections satisfying the release-notes CI gate.

**Checks performed** (all via `.venv/bin/python`, matching the contract's deterministic commands):
- `pytest tests/unit/test_claude_account_cli.py -q` → 22 passed
- `pytest tests/unit/test_usage.py -q` → 9 passed (token convention untouched, stays green)
- `pytest tests/ -q` → 862 passed, 17 skipped, 8 deselected, 1 xfailed (no failures/errors)
- `grep -RE '^\*\*Type:\*\* (patch|minor|major)' releases/_next/` → matched the new file
- `git status --porcelain` confirms only the four declared editable files were touched (`harness/usage.py`, `harness/agent.py`, `docs/decisions.md`, `tests/live`, and the contract/plan/pin files were not modified).

No blockers encountered; the implementation stayed within the declared scope and required outputs.
