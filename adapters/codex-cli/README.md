# Codex CLI adapter

Codex CLI's hook system (as of mid-2026 reporting) uses the same shape as
Claude Code's: a command hook reads JSON on stdin and signals back via
exit codes, over a smaller set of events (`SessionStart`,
`UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`,
`Stop`), command-type hooks only (no `agent`/`http`/`mcp_tool` types).

**Verify this against Codex CLI's current hook documentation before
relying on it** -- the exact JSON field names for a blocking decision on
`Stop` were not independently confirmed here, so the snippet below uses
plain exit codes, which are the part of the protocol that's most likely
correct: exit 0 = no issue, non-zero = flag it.

A representative config (adjust the config file name/location and the
exact hook syntax to match whatever Codex CLI documents at the time you
read this):

```toml
[hooks.Stop]
command = "python3"
args = ["scripts/check_pattern_consistency.py", "--format", "text", "--warn-only"]
```

`--warn-only` is used deliberately here rather than letting exit code 1
block the turn: since the exact blocking contract for Codex CLI's `Stop`
event isn't confirmed, start in observe-only mode, watch it run for a
while, and only remove `--warn-only` once you've confirmed what a
non-zero exit actually does in your Codex CLI version.
