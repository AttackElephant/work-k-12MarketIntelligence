The confidence block now shows the stated grade, weakest input and component grades, and console log records go to standard error. `.venv/bin/python -m pytest tests/ -q` passes: 868 passed, 17 skipped, 8 deselected, 1 xfailed. I did not run the contract's deterministic one-liner commands, the `git diff` protected-files check or the `grep` release-note check separately.

- **`run.py`**:
  - A new private helper, `_confidence_lines`, builds the block.
    - Axis ratings come first, in sorted order.
    - `grade: <value>` follows, then `weakest input: <grade>` with ` (<reference>)` when one is given.
    - Each component renders as `component <name>: <value>`, sorted by name.
  - If the response states no grade, or a grade with no value, the block shows the ratings and `grade: not stated by the data layer`. It shows no grade word, weakest-input line or component lines.
  - A missing weakest input or a missing component value renders as `not stated by the data layer`.
  - A non-grade axis with a dict or list value renders as `(not shown — unexpected structure)`, never as a repr.
  - The `render_answer` docstring and the module docstring's ADR-009 sentence are updated.
- **`logging_config.yaml`**: `handlers.console.stream` is now `ext://sys.stderr`, with a comment saying why. Nothing else changed.
- **`tests/unit/test_run_cli.py`**:
  - The old confidence test is replaced with one that holds to ADR-009 for a stated grade.
  - New tests cover L01 and VC-POC-13, read from the saved envelopes.
  - A parametrised test covers a missing grade and a missing `grade.value`, using deep copies of L01.
  - Text mode and `--json` each have a test that checks standard output and the log file under `tmp_path`. Both reconfigure logging afterwards.
- **`releases/_next/cli-confidence-grade-and-clean-stdout.md`**: new release note, type minor.

Before editing I searched for other users of `render_text`, the confidence heading and `ext://sys.stdout`. I found none that depend on the old output.

Nothing is blocked. A Modelled grade's backtest block is still out of scope, as the contract says.
