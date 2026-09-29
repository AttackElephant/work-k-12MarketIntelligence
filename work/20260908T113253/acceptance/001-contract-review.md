<!-- Derived view of 001-contract.yaml. The YAML is authoritative. Regenerate; never edit. sha256=738c54436db6cfb884c7cc34307700a5dd0d279fa429baad85e1168620337c76 -->

# Acceptance contract — work item 20260908T113253

contract_version: 0.1.0  ·  artifact_version: 1

## Observable behaviour
- **readme-reflects-provider** — README.md names `claude_account` (Claude CLI, subscription OAuth, model `claude-sonnet-4-6`) as the default provider per ADR-019 and `anthropic_sdk` as an explicit, potentially-metered opt-in; no statement claims the Anthropic SDK is the default and nothing leads with a stale provider/auth description drawn from architecture.md §8/§9.
- **readme-reflects-pipeline** — README.md summarises the implemented M1–M5 pipeline status honestly: the server layer is carried but unwired (O-2) and forecasting (VC-POC-14) is registered-but-gated (O-4), not part of MVP.
- **runbook-reflects-stages** — RUNBOOK.md names, in current terms, the deterministic PR-gate tier (pytest, zero network), the manual/nightly live tier (`python -m evals.live`) reporting pass@k and pass^k separately, the `--pressure` probe whose transcripts a human must read, the three demo modes and banner semantics, and the provider override (`HARNESS_MODEL_PROVIDER=anthropic_sdk`, potentially metered).
- **release-note-present** — Exactly one new releases/_next/<slug>.md exists carrying a `**Type:**` line and the mandatory `## Local deployment`, `## Manual steps` and `## Verification` (with a fenced bash block) sections, so the release-notes CI gate accepts the change; VERSION is untouched and no releases/v*.md is added.
- **no-behaviour-change** — run.py and demo.py change only help text, argparse `description`/`help=` and docstrings/comments where they name provider/model or eval stages; no runtime logic, exit code, control-flow or symbol behaviour changes.

## Required outputs
- `README.md` — The primary lagging surface: its top-level provider/model, evaluation-tier and pipeline-status prose must be reconciled to ADR-019, O-2 and O-4.
- `RUNBOOK.md` — The request names runbook guidance as a reconciliation target; operator prose for auth, the two eval tiers, --pressure, demo modes, provider override and the pin/checkpoint rules must be confirmed/tightened in current terms.
- `releases/_next/reconcile-doc-surfaces.md` — The release-notes CI gate (.github/workflows/pr-checks.yml release-notes job + .claude/hooks/require-release-notes.sh) requires exactly one new releases/_next/*.md with a Type line and the mandatory sections; the PR is blocked without it. Executor must use this slug.

## Deterministic criteria
- **full-test-suite** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/ -q`
  - expected: The declared verification suite (manifest k-12Harness.verify) exits zero with no failures and no errors; pre-existing xfail/xpass are unaffected.
  - machine-checked: yes
- **demo-properties-green** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_demo.py -q`
  - expected: The ADR-017 property tests over demo.py stay green after the help/docstring edits (no def render_*, renders through run.py, groups partition, single verbosity control).
  - machine-checked: yes
- **run-cli-green** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_run_cli.py -q`
  - expected: The CLI tests, including the require_credentials message and default_credentials source assertions, stay green after any run.py help edits.
  - machine-checked: yes
- **contract-pin-unchanged** in `k-12Harness`
  - command: `.venv/bin/python -m pytest tests/unit/test_contract_pin.py -q`
  - expected: The contract pin test stays green, proving no accidental contract/PINNED.json or CONTRACT_VERSION change (ADR-008); this is a docs-only WI with no contract bump.
  - machine-checked: yes
- **release-note-typed** in `k-12Harness`
  - command: `grep -RE '^\*\*Type:\*\* (patch|minor|major)' releases/_next/`
  - expected: The added releases/_next/ note declares a patch|minor|major Type line, as the release-notes gate requires; the base commit carries only releases/_next/.gitkeep, so exit 0 here depends on the change.
  - machine-checked: yes

## Must not change
- The require_credentials message strings in run.py asserted by tests/unit/test_run_cli.py (`claude setup-token`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`, and the `default_credentials` source reference) — kept intact, or their paired assertions updated in the same change.
- ADR-017 properties of demo.py: no `def render_*`, every answer renders via `cli.render_text(output)`, `demo.GROUPS`/`demo.GATED` partition the registry, and the single `os.environ.setdefault("LOG_LEVEL", ...)` control point with no direct `setLevel` (tests/unit/test_demo.py).
- Runtime behaviour of run.py and demo.py: no logic, exit code, argument parsing semantics or symbol behaviour change — only help/description/docstring/comment prose.
- contract/PINNED.json, tests/unit/test_contract_pin.py and settings.CONTRACT_VERSION: no contract version bump or re-pin in this WI.
- VERSION is not touched and no releases/v*.md is added (the release-notes gate refuses both); exactly one releases/_next/*.md is added.
- docs/architecture.md and docs/decisions.md (the pinned authority) are not edited; config/settings.py and the provider adapters (harness/claude_account_cli.py, harness/agent.py) carry no change.

## Failure behaviour
- A require_credentials message string in run.py is changed without updating tests/unit/test_run_cli.py. → test_run_cli.py fails; the executor must either keep the message strings intact or update the paired assertions in the same change.
- demo.py grows a `def render_*`, stops rendering through `cli.render_text`, or adds a second verbosity control. → tests/unit/test_demo.py fails; the demo must continue to render through run.py and keep the single LOG_LEVEL control.
- VERSION is bumped, a releases/v*.md is added, no releases/_next/*.md is added, more than one is added, or the note lacks the Type line or a mandatory section. → The release-notes CI gate and the local require-release-notes.sh hook deny the PR with an actionable reason; the change is blocked until the note conforms.

## Compatibility
- ADR-019: `claude_account` (Claude CLI, subscription OAuth, `claude-sonnet-4-6`) is the default provider and `anthropic_sdk` is explicit-and-potentially-metered; ADR-018 (Codex account) remains as history, superseded by ADR-019 — doc prose must state this and add no new superseding ADR.
- ADR-007: the live tier stays manual/nightly and never becomes a PR gate; the deterministic tier gates every PR at zero network cost.
- ADR-017: the demo has exactly three modes each announced by a tested banner and renders only through the product; doc prose must not contradict this.
- ADR-008 / ADR-015-3: the contract is pinned by hash and a bump is the loud four-step operation (re-pin, regenerate the prompt, regenerate the fixture bundle, re-point cases); the Anthropic SDK is pinned exactly alongside the framework as one compatibility surface — documented, not performed here.
- ADR-006: per-model cache minimums (CACHE_MIN_TOKENS) are re-verified on model upgrades and the cache regression check remains `cache_read_tokens > 0`.
- README.md's own precedence rule stands: docs/decisions.md is the source of truth on any conflict, so where architecture.md §8/§9 disagree with ADR-019 the surfaces adopt ADR-019.

## Compatibility baselines
_none_

## Contract requirements
- **scope-is-the-file-list** — The permitted change set is exactly required_outputs plus permitted_files. docs/architecture.md, docs/decisions.md, config/settings.py, the provider adapters, contract/PINNED.json, tests/unit/test_contract_pin.py, tests/unit/test_usage.py and harness/usage.py are outside scope.
- **decision-checkpoint-referent** — DECISION: which 'checkpoint compatibility rules' referent the doc surfaces reconcile to. Options: (a) the model-checkpoint/resume rule — the pinned account model (config/settings.py CLAUDE_ACCOUNT_MODEL) cannot resume a checkpoint produced by another model, per-model cache minimums re-verified on upgrade (ADR-006); (b) the eval-resume checkpoint (RUNBOOK §1 `--resume`); (c) the contract hash pin (ADR-008); (d) the SDK+framework pin surface (ADR-015-3). Neither Definition Pack reference defines a 'checkpoint' concept, so this is an owner choice. The rest of the contract is written on option (a). Affects: semantic checkpoint-rule-documented, observable_behaviour runbook-reflects-stages, and RUNBOOK.md / README.md content in required_outputs.
- **decision-release-note-type** — DECISION: whether the reconciliation edits the code entry points (run.py, demo.py) at all. Under the v1.2.0 path classes, any change to a root *.py behaviour path makes the release note `**Type:** minor`; a docs-only change (README, RUNBOOK, the note) is `**Type:** patch`. Options: (a) edit run.py/demo.py help/docstrings where they name provider/model/eval stages — release note Type: minor, run.py and demo.py are in the change set; (b) leave run.py/demo.py untouched — release note Type: patch. The contract is written on option (a) (the plan lists run.py/demo.py as in-scope surfaces) but keeps them in permitted_files so option (b) needs no restructuring; the release-note-typed deterministic accepts either Type. Affects: permitted_files (run.py, demo.py), observable_behaviour no-behaviour-change, semantic release-note-type-correct, deterministic release-note-typed.
- **gap-architecture-adr-conflict** — GAP: docs/architecture.md §8 (names the model plainly) and §9 (API-key/account-bearer auth) predate and conflict with docs/decisions.md ADR-018/019. Resolved by the repository's stated precedence (README.md: decisions.md is source of truth on any conflict): the surfaces adopt ADR-019; architecture.md is not edited here (its internal drift is out of scope).
- **assumption-readme-primary** — ASSUMPTION: README.md is the primary lagging surface; run.py, demo.py and RUNBOOK.md already largely reflect ADR-019, so reconciliation there is confirmation/tightening. The executor should diff each surface against the two pinned ADR sets before editing.
- **assumption-model-literal-source** — ASSUMPTION: the implemented pipeline/provider reality lives partly in code the manifest did not pin (config/settings.py, run.py, harness/claude_account_cli.py); where prose must name the pinned model it should point at config/settings.py as the single source of truth rather than restating a literal that can drift.

## Semantic criteria
- **provider-default-adopts-adr019** — README.md and RUNBOOK.md describe `claude_account` (Claude CLI, subscription OAuth, `claude-sonnet-4-6`) as the default provider and `anthropic_sdk` as explicit/potentially-metered, with no residual claim that the Anthropic SDK is the default and no stale auth description contradicting ADR-019.
- **pipeline-status-honest** — README.md's pipeline summary states the M1–M5 reality without overstating it: the server layer is carried but unwired (O-2) and forecasting is registered-but-gated (O-4), never presented as live or MVP.
- **eval-stages-named** — RUNBOOK.md names the deterministic PR-gate tier, the manual/nightly live tier reporting pass@k and pass^k separately, the `--pressure` probe whose transcripts must be read by a human, and the three demo modes with banner semantics — consistent with ADR-007 and ADR-017.
- **checkpoint-rule-documented** — Per decision-checkpoint-referent option (a): RUNBOOK.md/README.md state that the pinned account model (config/settings.py CLAUDE_ACCOUNT_MODEL) cannot resume a checkpoint produced by another model and that per-model cache minimums are re-verified on model upgrades (ADR-006). If the owner selects another referent this criterion changes.
- **release-note-type-correct** — Per decision-release-note-type: the new releases/_next note's `**Type:**` matches the highest v1.2.0 path class of the files actually changed — `minor` if any behaviour path (run.py/demo.py) is edited (option a), `patch` if only docs are edited (option b).
- **no-authority-doc-edits** — docs/architecture.md and docs/decisions.md are unchanged; the reconciliation adds no behaviour claim beyond what those ADRs state.

## Out of scope
- Editing docs/architecture.md or docs/decisions.md, or adding a superseding ADR (the append-only authority; internal drift is a GAP, not an edit target here).
- Any contract version bump, re-pin, or edit to contract/PINNED.json or tests/unit/test_contract_pin.py.
- Any change to config/settings.py, harness/claude_account_cli.py, harness/agent.py or provider-selection/eval mechanics (behaviour).
- Running the live tier or any network/credentialed run; a docs-only change needs none.
- Editing harness/usage.py report help text or tests/unit/test_usage.py (not planned).
- Redesigning the run.py/demo.py CLI surface or the evaluation tiers beyond reconciling their descriptive prose.

## Base-commit dry run

Every deterministic criterion run once at `k-12Harness` @ e4141843ae5a, before any change (2026-09-29T03:15:57Z). A flag means the result cannot depend on the change.

- **full-test-suite** — ran, exit 0 on the base
- **demo-properties-green** — ran, exit 0 on the base
- **run-cli-green** — ran, exit 0 on the base
- **contract-pin-unchanged** — ran, exit 0 on the base
- **release-note-typed** — ran, exit 1 on the base

_no criterion flagged_
