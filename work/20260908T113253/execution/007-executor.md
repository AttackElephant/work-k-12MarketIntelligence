## Summary

**Files changed:**
- `README.md` — Fixed the "Live-eval checkpoints" section to remove the contradictory claim that a mismatched checkpoint "starts a fresh run." It now states the evaluator **refuses** an incompatible `--resume` with an error, never starts a fresh run on its own, and the operator must explicitly start a separate fresh run (drop `--resume`).
- `RUNBOOK.md` — Tightened the "Checkpoint compatibility" note to match README's wording exactly: on mismatch the evaluator refuses `--resume` with an error, never starts a fresh run itself, and the operator must start a separate fresh run.
- `releases/_next/reconcile-doc-surfaces.md` — unchanged (already present from the prior candidate; verified it still satisfies the gate).

No other files were touched, per the guidance to "change nothing else."

**Checks performed:**
- `.venv/bin/python -m pytest tests/ -q` → `862 passed, 17 skipped, 8 deselected, 1 xfailed` (exit 0, no failures/errors).
- `.venv/bin/python -m pytest tests/unit/test_contract_pin.py -q` → `3 passed` (exit 0).
- `grep -RE '^\*\*Type:\*\* patch' releases/_next/` → matched `releases/_next/reconcile-doc-surfaces.md:**Type:** patch` (exit 0).

**Unresolved requirements:** none — README.md and RUNBOOK.md now agree on the live-eval resume/checkpoint rule (refuse-with-error; operator starts a separate fresh run), consistent with DECIDED C1 and the human guidance.
