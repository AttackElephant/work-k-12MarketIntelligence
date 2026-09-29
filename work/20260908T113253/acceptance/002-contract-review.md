<!-- Derived view of 002-contract.yaml. The YAML is authoritative. Regenerate; never edit. sha256=ea24050b01aea3141c271e692fe40e6f09f39cb154005c1d2a1867d7fbf78de5 -->

# Acceptance contract — work item 20260908T113253

contract_version: 0.1.0  ·  artifact_version: 2

## Observable behaviour
- **readme-reflects-provider** — README.md names `claude_account` (Claude CLI, subscription OAuth, model `claude-sonnet-4-6`) as the default provider per ADR-019 and `anthropic_sdk` as an explicit, potentially-metered opt-in; no statement claims the Anthropic SDK is the default and nothing leads with a stale provider/auth description drawn from architecture.md §8/§9.
- **readme-reflects-pipeline** — README.md summarises the implemented M1–M5 pipeline status honestly: the server layer is carried but unwired (O-2) and forecasting (VC-POC-14) is registered-but-gated (O-4), not part of MVP.
- **runbook-reflects-stages** — RUNBOOK.md names, in current terms, the deterministic PR-gate tier (pytest, zero network), the manual/nightly live tier (`python -m evals.live`) reporting pass@k and pass^k separately, the `--pressure` probe whose transcripts a human must read, the three demo modes and banner semantics, the provider override (`HARNESS_MODEL_PROVIDER=anthropic_sdk`, potentially metered), and the live-eval resume/checkpoint compatibility rules as implemented in evals/live.py (a saved checkpoint resumes only when report format version, contract version, model and stage all match and the saved case selection still exists) — see decision-checkpoint-referent (DECIDED C1).
- **release-note-present** — Exactly one new releases/_next/<slug>.md exists carrying a `**Type:** patch` line and the mandatory `## Local deployment`, `## Manual steps` and `## Verification` (with a fenced bash block) sections, so the release-notes CI gate accepts the change; VERSION is untouched and no releases/v*.md is added.
- **no-behaviour-change** — Per decision-release-note-type (DECIDED C2), run.py and demo.py are not modified at all — nothing in them is stale, they already reflect ADR-019 and ADR-017 — so no help text, argparse `description`/`help=`, docstring, comment, runtime logic, exit code, control-flow or symbol change occurs in them; the only files changed are README.md, RUNBOOK.md and the new release note.

## Required outputs
- `README.md` — The primary lagging surface: its top-level provider/model, evaluation-tier and pipeline-status prose must be reconciled to ADR-019, O-2 and O-4, and it states the live-eval resume/checkpoint compatibility rules (DECIDED C1).
- `RUNBOOK.md` — The request names runbook guidance as a reconciliation target; operator prose for auth, the two eval tiers, --pressure, demo modes, provider override and the live-eval resume/checkpoint compatibility rules (evals/live.py, DECIDED C1) must be confirmed/tightened in current terms.
- `releases/_next/reconcile-doc-surfaces.md` — The release-notes CI gate (.github/workflows/pr-checks.yml release-notes job + .claude/hooks/require-release-notes.sh) requires exactly one new releases/_next/*.md with a Type line and the mandatory sections; the PR is blocked without it. Executor must use this slug; the Type is patch (DECIDED C2).

## Deterministic criteria
- **full-test-suite** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/ -q`
  - expected: The declared verification suite (manifest k-12Harness.verify) exits zero with no failures and no errors; pre-existing xfail/xpass are unaffected. This covers tests/unit/test_run_cli.py and tests/unit/test_demo.py, which must stay green with run.py/demo.py unchanged.
  - machine-checked: yes
- **contract-pin-unchanged** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_contract_pin.py -q`
  - expected: The contract pin test stays green, proving no accidental contract/PINNED.json or CONTRACT_VERSION change (ADR-008); this is a docs-only WI with no contract bump.
  - machine-checked: yes
- **release-note-typed** in `k-12Harness`
  - command: `grep -RE '^\*\*Type:\*\* patch' releases/_next/`
  - expected: The added releases/_next/ note declares `**Type:** patch`, as DECIDED C2 requires for a docs-only change under the v1.2.0 path classes; the base commit carries only releases/_next/.gitkeep, so exit 0 here depends on the change.
  - machine-checked: yes

## Must not change
- run.py is not edited in this WI (DECIDED C2): its require_credentials message strings (`claude setup-token`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`, and the `default_credentials` source reference) and the paired assertions in tests/unit/test_run_cli.py stay intact.
- demo.py is not edited in this WI (DECIDED C2): its ADR-017 properties — no `def render_*`, every answer renders via `cli.render_text(output)`, `demo.GROUPS`/`demo.GATED` partition the registry, and the single `os.environ.setdefault("LOG_LEVEL", ...)` control point with no direct `setLevel` (tests/unit/test_demo.py) — stay intact.
- Runtime and prose of run.py and demo.py: no logic, exit code, argument-parsing semantics, help/description/docstring/comment or symbol change — they are not modified at all in this WI.
- contract/PINNED.json, tests/unit/test_contract_pin.py and settings.CONTRACT_VERSION: no contract version bump or re-pin in this WI.
- VERSION is not touched and no releases/v*.md is added (the release-notes gate refuses both); exactly one releases/_next/*.md is added.
- docs/architecture.md and docs/decisions.md (the pinned authority) are not edited; config/settings.py and the provider adapters (harness/claude_account_cli.py, harness/agent.py) carry no change; evals/live.py (the checkpoint-rule referent) is documented, not edited.

## Failure behaviour
- run.py, demo.py, or any file outside required_outputs (README.md, RUNBOOK.md, releases/_next/reconcile-doc-surfaces.md) is modified. → The edit leaves the declared permitted file set; it is a scope violation the controller refuses until a human records a scope defect and a new contract version widens scope. run.py and demo.py are deliberately out of scope (DECIDED C2) because nothing in them is stale.
- VERSION is bumped, a releases/v*.md is added, no releases/_next/*.md is added, more than one is added, or the note lacks the `**Type:** patch` line or a mandatory section. → The release-notes CI gate and the local require-release-notes.sh hook deny the PR with an actionable reason; the change is blocked until the note conforms.

## Compatibility
- ADR-019: `claude_account` (Claude CLI, subscription OAuth, `claude-sonnet-4-6`) is the default provider and `anthropic_sdk` is explicit-and-potentially-metered; ADR-018 (Codex account) remains as history, superseded by ADR-019 — doc prose must state this and add no new superseding ADR.
- ADR-007: the live tier stays manual/nightly and never becomes a PR gate; the deterministic tier gates every PR at zero network cost.
- ADR-017: the demo has exactly three modes each announced by a tested banner and renders only through the product; doc prose must not contradict this.
- ADR-008 / ADR-015-3: the contract is pinned by hash and a bump is the loud four-step operation (re-pin, regenerate the prompt, regenerate the fixture bundle, re-point cases); the Anthropic SDK is pinned exactly alongside the framework as one compatibility surface — documented, not performed here.
- ADR-006: per-model cache minimums (CACHE_MIN_TOKENS) are re-verified on model upgrades and the cache regression check remains `cache_read_tokens > 0`.
- The live-eval resume/checkpoint compatibility rules as implemented in evals/live.py (report format version, contract version, model and stage must match, and the saved case selection must still exist) are the referent the doc surfaces reconcile to (DECIDED C1); doc prose describes them and adds no new resume rule.
- README.md's own precedence rule stands: docs/decisions.md is the source of truth on any conflict, so where architecture.md §8/§9 disagree with ADR-019 the surfaces adopt ADR-019.

## Compatibility baselines
_none_

## Contract requirements
- **scope-is-the-file-list** — The permitted change set is exactly required_outputs (README.md, RUNBOOK.md, releases/_next/reconcile-doc-surfaces.md); permitted_files is empty. run.py, demo.py, tests/unit/test_run_cli.py, tests/unit/test_demo.py, docs/architecture.md, docs/decisions.md, config/settings.py, the provider adapters, evals/live.py, contract/PINNED.json, tests/unit/test_contract_pin.py, tests/unit/test_usage.py and harness/usage.py are outside scope.
- **decision-checkpoint-referent** — DECIDED (C1): the 'checkpoint compatibility rules' the doc surfaces reconcile to are the live-eval resume rules as implemented in evals/live.py — a saved checkpoint may be resumed only when the report format version, the contract version, the model and the stage all match, and the saved case selection must still exist. The doc surfaces document this referent, not the model-checkpoint/ADR-006 upgrade rule, the contract hash pin (ADR-008), or the SDK+framework pin surface (ADR-015-3). Governs semantic checkpoint-rule-documented, observable_behaviour runbook-reflects-stages, and RUNBOOK.md / README.md content in required_outputs.
- **decision-release-note-type** — DECIDED (C2): run.py and demo.py are not edited — nothing in them is stale; they already reflect ADR-019 and ADR-017 — so they leave the change set and permitted_files, and the reconciliation is docs-only (README.md, RUNBOOK.md and the new release note). Under the v1.2.0 path classes a docs-only change (README, RUNBOOK, releases) is `**Type:** patch`; the release note is patch. Governs permitted_files (now empty of run.py/demo.py), observable_behaviour no-behaviour-change, semantic release-note-type-correct and deterministic release-note-typed.
- **gap-architecture-adr-conflict** — GAP: docs/architecture.md §8 (names the model plainly) and §9 (API-key/account-bearer auth) predate and conflict with docs/decisions.md ADR-018/019. Resolved by the repository's stated precedence (README.md: decisions.md is source of truth on any conflict): the surfaces adopt ADR-019; architecture.md is not edited here (its internal drift is out of scope).
- **assumption-readme-primary** — ASSUMPTION: README.md is the primary lagging surface; run.py and demo.py already reflect ADR-019/ADR-017 and carry nothing stale (DECIDED C2), so they are not edited, and RUNBOOK.md reconciliation is confirmation/tightening. The executor should diff README.md and RUNBOOK.md against the two pinned ADR sets and evals/live.py before editing.
- **assumption-model-literal-source** — ASSUMPTION: the implemented pipeline/provider reality lives partly in code the manifest did not pin (config/settings.py, run.py, harness/claude_account_cli.py); where prose must name the pinned model it should point at config/settings.py as the single source of truth rather than restating a literal that can drift.

## Semantic criteria
- **provider-default-adopts-adr019** — README.md and RUNBOOK.md describe `claude_account` (Claude CLI, subscription OAuth, `claude-sonnet-4-6`) as the default provider and `anthropic_sdk` as explicit/potentially-metered, with no residual claim that the Anthropic SDK is the default and no stale auth description contradicting ADR-019.
- **pipeline-status-honest** — README.md's pipeline summary states the M1–M5 reality without overstating it: the server layer is carried but unwired (O-2) and forecasting is registered-but-gated (O-4), never presented as live or MVP.
- **eval-stages-named** — RUNBOOK.md names the deterministic PR-gate tier, the manual/nightly live tier reporting pass@k and pass^k separately, the `--pressure` probe whose transcripts must be read by a human, and the three demo modes with banner semantics — consistent with ADR-007 and ADR-017.
- **checkpoint-rule-documented** — Per decision-checkpoint-referent (DECIDED C1): RUNBOOK.md/README.md document the live-eval resume compatibility rules as implemented in evals/live.py — a saved checkpoint may be resumed only when the report format version, the contract version, the model and the stage all match and the saved case selection still exists — and do not misdescribe the referent as the model-checkpoint/ADR-006 upgrade rule, the contract hash pin, or the SDK+framework pin surface.
- **release-note-type-correct** — Per decision-release-note-type (DECIDED C2): the new releases/_next note's `**Type:**` is `patch`, because only docs (README.md, RUNBOOK.md and the note) change and run.py/demo.py are untouched.
- **no-authority-doc-edits** — docs/architecture.md and docs/decisions.md are unchanged; the reconciliation adds no behaviour claim beyond what those ADRs and evals/live.py state.

## Out of scope
- Editing run.py or demo.py: nothing in them is stale (DECIDED C2), so they are not modified in this WI.
- Editing docs/architecture.md or docs/decisions.md, or adding a superseding ADR (the append-only authority; internal drift is a GAP, not an edit target here).
- Any contract version bump, re-pin, or edit to contract/PINNED.json or tests/unit/test_contract_pin.py.
- Any change to config/settings.py, harness/claude_account_cli.py, harness/agent.py, evals/live.py or provider-selection/eval mechanics (behaviour); evals/live.py's resume rules are documented, not changed.
- Running the live tier or any network/credentialed run; a docs-only change needs none.
- Editing harness/usage.py report help text or tests/unit/test_usage.py (not planned).
- Redesigning the run.py/demo.py CLI surface or the evaluation tiers beyond reconciling their descriptive prose.

## Base-commit dry run

Every deterministic criterion run once at `k-12Harness` @ e4141843ae5a, before any change (2026-09-29T07:18:16Z). A flag means the result cannot depend on the change.

- **full-test-suite** — ran, exit 0 on the base
- **contract-pin-unchanged** — ran, exit 0 on the base
- **release-note-typed** — ran, exit 1 on the base

_no criterion flagged_
