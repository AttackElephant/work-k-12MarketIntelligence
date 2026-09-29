# Implementation Plan

## Bounded objective
Reconcile the k-12Harness *documentation and help surfaces* — entry-point CLI help/docstrings (`run.py`, `demo.py`), operational guidance (`RUNBOOK.md`, `README.md`), and release documentation (`releases/`) — so their prose matches the implemented reality already recorded in the two Definition Pack references: (1) the implemented pipeline status (M1–M5 build order and what is/isn't wired), (2) the explicit evaluation stages (deterministic PR-gate tier, manual/nightly live tier reporting pass@k/pass^k, the `--pressure` probe, and the three demo modes), (3) the pinned model / default provider, and (4) the checkpoint/pin compatibility rules. This is a documentation-reconciliation work item: **no runtime behaviour changes** — only help text, docstrings, runbook/README prose, and a release note. Provider-selection logic, eval mechanics, and symbol behaviour stay as implemented.

## Definition Pack interpretation
Authority boundary = the two references pinned in `manifest.yaml`: `docs/architecture.md` (sha256 d20e51…) and `docs/decisions.md` (sha256 18d2cce…). Both were provided in full. Salient facts the doc surfaces must be reconciled *to*:
- **Provider / model (decisions.md ADR-018 → ADR-019).** `claude_account` is the default provider, invoking `claude --print` under the inherited `CLAUDE_CODE_OAUTH_TOKEN`, with built-in tools/MCP/session persistence disabled and metered env vars stripped. `anthropic_sdk` is available *only* by explicit `HARNESS_MODEL_PROVIDER=anthropic_sdk` and is documented as potentially metered. ADR-019 supersedes ADR-018; ADR-018 remains as history. Account plans expose no per-request USD, so the harness reports account-plan usage and retains only non-financial step/tool-call/wall-clock ceilings. `settings.py` confirms the account route is pinned to `claude-sonnet-4-6` (`CLAUDE_ACCOUNT_MODEL`), an experimental override that cannot resume a checkpoint produced by another model.
- **Evaluation stages (decisions.md ADR-007, ADR-013–015, ADR-017; O-10).** Deterministic tier gates every PR at zero network cost; live tier (`python -m evals.live`, `tests/live/`) is manual/nightly and reports pass@k and pass^k *separately*; `--pressure` (O-10) appends red-team phrasings and writes transcripts for a human to read (it does not grade). `evals/replay.py` runs the full offline production path. Demo (ADR-017) has three modes — `--replay`, `--stub`, and neither-flag (live) — each announced by a tested banner.
- **Pipeline status (architecture.md §12; decisions.md O-2, O-4).** M1–M5 milestones; server layer (`server/`) is carried but unwired (O-2); forecasting is registered-but-gated (VC-POC-14, O-4 resolved), not part of MVP.
- **Pin / compatibility rules (decisions.md ADR-008, ADR-015 decision 3; ADR-006).** The contract is pinned by hash (`contract/PINNED.json`); a bump is a loud four-step operation (re-pin, regenerate the prompt, regenerate the fixture bundle, re-point cases). The Anthropic SDK is pinned exactly alongside the framework as one compatibility surface (ADR-015-3). Per-model cache minimums (`CACHE_MIN_TOKENS`) are re-verified on model upgrades (ADR-006).

## Current implementation
Established from the authority docs and the provided bodies of `run.py`, `demo.py`, `RUNBOOK.md`, `README.md`, `config/settings.py`, `releases/_template.md`, `releases/v1.2.0.md` and `tests/unit/test_run_cli.py`:
- Entry points present: `run.py` (CLI), `demo.py` (walked demo), `evals/live.py` (`python -m evals.live`), `harness/usage.py` (`python -m harness.usage report`), `prompts/build_prompt.py` (`--capture`).
- `run.py`, `demo.py`, `RUNBOOK.md`, `config/settings.py` are already substantially aligned with ADR-019 (claude_account default, `claude-sonnet-4-6` pin, account-plan usage, three demo modes, live-tier pass@k/pass^k, `--pressure`, contract/SDK pin adoption). `README.md` is the surface most likely to lag: it still leads on "A PydanticAI agent" and gives a briefer provider paragraph that could conflict with the pipeline reality.
- `RUNBOOK.md` already states contract **v0.3.0**, the four-step bump, and that live has never run against a real model — reconciliation here is confirming/tightening, not rewriting.
- Enforcing checks in the inventory: `tests/unit/test_run_cli.py` (rendering, exit codes, clarification round-trips, credential messages; greps `run.py` for `default_credentials` — does **not** pin argparse help verbatim), `tests/unit/test_demo.py` (ADR-017 properties: no `render_*` in demo.py, banner, `GROUPS` partition), `tests/unit/test_usage.py`, the release-note gate (`.claude/hooks/require-release-notes.sh` + `.github/workflows/pr-checks.yml` `release-notes` job), and the contract pin test `tests/unit/test_contract_pin.py`.
- **Suspected drift the request targets:** architecture.md §8 still names the model plainly and §9 describes API-key/account-bearer auth — text predating ADR-018/019's move to the Claude CLI subscription provider. Any README/RUNBOOK statement echoing §8/§9 is the likely stale-model / stale-auth wording to reconcile.

## Proposed changes
Each file below is in scope; any enforcing file not named here is out of scope and the executor may not implement around it.
- **root1:README.md** — Reconcile the top-level description of provider/model, evaluation tiers, and pipeline status: state `claude_account` (Claude CLI, subscription OAuth, `claude-sonnet-4-6`) as the default provider per ADR-019, `anthropic_sdk` as explicit-and-metered, remove/qualify any stale "anthropic SDK default" claim, and summarise the M1–M5 status honestly (server unwired per O-2, forecasting gated per O-4). No behaviour claims beyond the ADRs.
- **root1:RUNBOOK.md** — Confirm/tighten operator guidance so it names, in current terms: auth setup (`CLAUDE_CODE_OAUTH_TOKEN` inherited, `SHARED_ENV_FILE` machine-level secrets, ADR-011/019), the deterministic tier (pytest, PR gate, zero network), the live tier (`python -m evals.live`, manual/nightly, pass@k and pass^k reported separately), the `--pressure` probe (transcripts must be read by a human), the three demo modes and banner semantics, the provider override (`HARNESS_MODEL_PROVIDER=anthropic_sdk`, potentially metered), and the contract/SDK pin-adoption rules (ADR-008 four-step bump; ADR-015-3 SDK+framework as one pinned surface; the checkpoint-model note from settings).
- **root1:run.py** — Update argparse `help=`/`description` and the module docstring only where they name provider/model or eval stages; **no logic change**, and the `require_credentials` message wording must remain intact (asserted by `test_run_cli.py`). `test_run_cli.py` does not pin help strings, so help edits are safe.
- **root1:demo.py** — Align mode/help/docstring prose with ADR-017's three modes and banner semantics; must not introduce a `def render_*` (ADR-017 property, tested) and must not restate rendering rules.
- **root1:releases/_next/<slug>.md** — New release note copied from `root1:releases/_template.md`, `**Type:** patch` (docs-only, non-behaviour under the v1.2.0 path classes: `docs/**`, `releases/**`, root `*.md` are patch; but `run.py`/`demo.py` are behaviour paths — see GAP on classification). Required by the release-note gate; must gain a correctly-typed entry with the mandatory sections or CI blocks the change.

Enforcing checks that must be updated in lockstep (in scope):
- **root1:tests/unit/test_run_cli.py** — verified not to assert on argparse help text; must stay green. Only if the executor changes any `require_credentials` message string must the matching assertions here be updated.
- **root1:tests/unit/test_demo.py** — must continue to hold the ADR-017 properties (no `render_*` in `demo.py`, banner present, `demo.GROUPS` partition) and gain/adjust any assertion over edited mode/help text.
- **root1:tests/unit/test_usage.py** — in scope only if `harness/usage.py` report help text is edited (not planned); otherwise leave untouched.

Explicitly out of scope (do not edit): `docs/architecture.md` and `docs/decisions.md` (the authority; internal drift is a GAP, not an edit target here), `config/settings.py` and provider adapters (behaviour), `contract/PINNED.json` and `tests/unit/test_contract_pin.py` (no contract bump in this WI).

## Verification approach
- Run the declared deterministic suite: `.venv/bin/python -m pytest tests/ -q` (manifest `k-12Harness.verify`) — must stay green, in particular `tests/unit/test_run_cli.py`, `tests/unit/test_demo.py`, `tests/unit/test_usage.py`, and `tests/unit/test_contract_pin.py` (proves no accidental pin/behaviour change).
- Confirm the release-note gate passes: `.claude/hooks/require-release-notes.sh` and the `pr-checks` `release-notes` job accept the new `releases/_next/<slug>.md` (correct `**Type:**`, all required sections present).
- Deterministic doc-consistency acceptance (analogue of sibling WIs' `usage-convention-green` / `release-note-typed` dry-run checks): assert README/RUNBOOK no longer state a provider/model contradicting ADR-019 and that they name the deterministic tier, the live pass@k/pass^k tier, `--pressure`, and the three demo modes.
- No live-tier or network run is required or appropriate for a docs-only change.

## Risks, assumptions and gaps
- **Risk:** overstating status (claiming the server layer or forecasting is live) would re-introduce drift; guidance must mirror O-2 (server unwired) and O-4 (forecasting gated).
- **Risk:** changing a `require_credentials` message string would break `test_run_cli.py`; keep those messages intact or update the paired assertions in the same change.
- **Assumption:** README is the primary lagging surface; RUNBOOK/run.py/demo.py already largely reflect ADR-019, so reconciliation there is confirmation/tightening rather than rewriting. The executor should diff each surface against the two ADR sets before editing.
- **Assumption:** no contract version bump is involved, so `PINNED.json`/pin test stay untouched; where prose must name the pinned model, it should point at `config/settings.py` (single source of truth) rather than restating a literal that can drift.

## Planning gaps
GAP: docs/architecture.md §8 ("Model: claude-sonnet-4-6") and §9 (API-key/account-bearer auth) conflict with docs/decisions.md ADR-019 (claude_account / Claude-CLI subscription OAuth as default, ADR-018 superseded) — the authoritative default-provider/model statement the doc surfaces must adopt is unresolved between these two Definition Pack sections.
GAP: request.md names "checkpoint compatibility rules" but neither Definition Pack reference defines a "checkpoint" concept; the referent is ambiguous between model-checkpoint upgrade re-verification (config/settings.py CLAUDE_ACCOUNT_MODEL note + ADR-006 CACHE_MIN_TOKENS), the eval-resume checkpoint (RUNBOOK §1 `--resume`), the contract hash pin (ADR-008), and the SDK+framework pin surface (ADR-015-3) — requires definition escalation before the checkpoint-compatibility wording can be authored.
GAP: manifest.yaml pins only docs/architecture.md and docs/decisions.md as references, but the request's "implemented pipeline status" and current provider default live in code the manifest did not capture (config/settings.py, run.py, harness/claude_account_cli.py); the authoritative source for "implemented" reality is therefore only partially pinned by the Definition Pack.
GAP: releases/v1.2.0.md path classes make root `*.py` entry points behaviour (minor) while docs/releases are patch; a single release note editing both run.py/demo.py and README/RUNBOOK spans two classes, and the note's `**Type:**` for a docs-and-help change is unresolved between the request (docs reconciliation) and the declared classification rule.

## Size estimate
A bounded documentation-and-help reconciliation touching ~5 surface files plus one release note, with two enforcing test files held green (updated only if paired strings change). No runtime behaviour change. It fits the objective and does not need a split. The single genuine blocker is the undefined "checkpoint compatibility rules" term, which should be resolved by definition escalation rather than by inventing a rule.

```yaml
size_estimate:
  files: 6
  scale: medium          # small | medium | large
  subsystems: [docs, cli, releases, tests]
```
