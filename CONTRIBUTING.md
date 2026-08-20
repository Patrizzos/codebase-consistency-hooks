# Contributing

## Adding a check

1. **Write the function** in `scripts/check_pattern_consistency.py`:
   ```python
   def check_<name>(text: str) -> str | None:
       # Return a short label if the check applies to this file,
       # None if it doesn't (not "no finding" — "not applicable").
   ```
   Keep it regex-based and dependency-free (stdlib only).

2. **Register it** in the `CHECKS` list:
   ```python
   ("name", check_name, {".py", ".ts"})
   ```
   The extension set scopes the check to file types where the regex makes sense.
   Too broad = false positives. Too narrow = missed coverage.

3. **Add it to `consistency.json`** under `"checks"` so teams can disable it:
   ```json
   "checks": { "your_check_name": true }
   ```

4. **Test it** against a throwaway git repo:
   - Create 3+ files using one convention.
   - Create 1 file breaking it.
   - Run `python3 scripts/check_pattern_consistency.py --format text`.
   - Confirm: flags only the divergent file, stays silent once fixed.

## Adding an adapter

1. Create `adapters/<harness>/` with a `README.md`.
2. The adapter's only job: invoke `python3 scripts/check_pattern_consistency.py`
   with the appropriate `--format` flag and act on the exit code or JSON output.
3. Use `--format text` or `--format json` for new harnesses unless the harness
   has a native hook protocol (like Claude Code's Stop event JSON contract).
4. Default to `--warn-only` until you've confirmed the blocking contract of
   the target harness — then document when to remove it.

## Principle

Engine and adapters stay separated. New tool support = new adapter directory,
not a reimplementation of the analysis. The engine doesn't know or care what
called it.
