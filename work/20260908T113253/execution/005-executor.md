All checks pass and only the declared editable files changed.

## Summary

**Files changed:**
- `README.md` — states `claude_account` (Claude CLI, subscription OAuth, `claude-sonnet-4-6`) as the ADR-019 default provider and `anthropic_sdk` as an explicit, potentially metered opt-in; adds a "Pipeline status (M1–M5)" section stating the server layer is unwired (O-2) and forecasting is registered-but-gated (O-4); adds a "Live-eval checkpoints" section documenting the `evals/live.py` resume-compatibility rule (report format version, contract version, model, stage must match, and the saved case selection must still exist).
- `RUNBOOK.md` — labels §0 as the deterministic PR-gate tier (pytest, zero network) and confirms the three demo modes/banner semantics; labels the live section as the manual/nightly live tier reporting pass@k/pass^k separately; clarifies the `HARNESS_MODEL_PROVIDER=anthropic_sdk` override is explicit and potentially metered; adds an explicit checkpoint-compatibility paragraph next to the `--resume` command; the `--pressure` probe section was already present and unchanged in substance.
- `releases/_next/reconcile-doc-surfaces.md` — new release note, `**Type:** patch`, with all mandatory sections (`## Local deployment`, `## Manual steps`, `## Verification` with a fenced bash block).

**Not touched:** `run.py`, `demo.py`, `docs/architecture.md`, `docs/decisions.md`, `contract/PINNED.json`, `VERSION`, and no `releases/v*.md` was added — per `git status`, only the three declared files are modified/added.

**Checks run:**
- `.venv/bin/python -m pytest tests/ -q` → `862 passed, 17 skipped, 8 deselected, 1 xfailed` (0 failures/errors).
- `.venv/bin/python -m pytest tests/unit/test_contract_pin.py -q` → `3 passed`.
- `grep -RE '^\*\*Type:\*\* patch' releases/_next/` → matched the new note, exit 0.

No unresolved requirements; all declared observable behaviours, semantic criteria and deterministic checks in the contract are addressed within the permitted file set.
