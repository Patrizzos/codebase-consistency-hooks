# Cursor adapter

The same harness-agnostic engine works in Cursor via its Rules system.
Two file formats are provided; use whichever matches your Cursor version.

## Modern Cursor — rules directory (recommended)

```bash
mkdir -p .cursor/rules
cp adapters/cursor/consistency-check.mdc .cursor/rules/consistency-check.mdc
```

Cursor picks up `.mdc` files in `.cursor/rules/` automatically. The rule
fires on the glob set in the frontmatter (`**/*.py`, `**/*.ts`, etc.).

## Legacy Cursor — .cursorrules

```bash
cp adapters/cursor/.cursorrules .cursorrules
```

If `.cursorrules` already exists in your repo, append the file contents
rather than replacing.

## What it does

After any matching file edit, Cursor includes the rule in its context
window. The model is instructed to run the consistency check and address
findings before completing the task. The check itself runs identically to
the git and CI adapters — same engine, same exit codes, same output.

## Enforcement note

Cursor rules are advisory: the model reads them but they don't hard-block
the way a pre-commit hook does. Pair with the git adapter for real
enforcement at commit/push time.
