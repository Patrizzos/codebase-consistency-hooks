# VS Code adapter

Exposes the consistency checker as VS Code tasks (Terminal → Run Task).
Works regardless of what tool or extension wrote the code.

## Setup

```bash
mkdir -p .vscode
cp adapters/vscode/tasks.json .vscode/tasks.json
```

If `.vscode/tasks.json` already exists, merge the `tasks` array rather than
replacing the file.

## Tasks provided

| Task | Output | Blocking |
|---|---|---|
| `check` | text — human-readable findings | yes (exit 1) |
| `check (JSON)` | full structured output | yes (exit 1) |
| `emit SARIF` | writes `consistency-results.sarif` | no (`--warn-only`) |
| `check since branch base` | mirrors what CI checks on a PR | yes (exit 1) |

## SARIF in VS Code

Install the **SARIF Viewer** extension (`MS-SarifVSCode.sarif-viewer`) to
browse findings from the SARIF task inline in the editor, with links
directly to the flagged lines.

## Keyboard shortcut (optional)

In `keybindings.json`:
```json
{
  "key": "ctrl+shift+k",
  "command": "workbench.action.tasks.runTask",
  "args": "codebase-consistency: check"
}
```
