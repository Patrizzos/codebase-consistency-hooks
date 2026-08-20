# codebase-consistency-hooks

A pattern-convergence checker: after code changes, it looks for similarly
structured solutions already in the codebase and flags where the new code
diverges from the dominant existing convention — so differences wind down
over time instead of accumulating.

Works with Claude Code, Codex CLI, Cursor, git hooks, GitHub Actions, and
VS Code. One engine. Thin adapters. Zero dependencies.

---

## No tooling? Just use the prompt.

The full concept is also available as a single structured prompt in [`PROMPT.md`](./PROMPT.md). Copy it into any agent, chat interface, or AI coding assistant that has read access to your repository — no installation, no git hooks, no Python required. It encodes the same family-detection logic, the same seven checks, the same flagging threshold, and the same output format as the engine.

The Python engine and adapters exist for automation and enforcement. The prompt exists for everything else.

---

## How it works

```
git diff  →  changed files  →  find structural family  →  run convention checks
                                 (token-shingle jaccard)    (regex, ext-scoped)
                                                                    ↓
                                              dominant convention in family?
                                              changed file matches? → silent
                                              changed file diverges? → flag it
                                                                    ↓
                                              optional: LLM enrichment (HTTP)
                                                                    ↓
                                              output: text | json | sarif |
                                                      claude-code-stop |
                                                      claude-code-context
```

**What "structural family" means:** identifiers are normalised to a
placeholder before comparing, so `get_user()` and `get_payment()` score as
structurally identical even though nothing textually matches. This catches
"same shape, different names" — the normal case for similarly-structured
solutions — rather than requiring literal near-duplicates.

**When it flags:** only when ≥ 60% of the family (configurable) uses a
different convention than the changed file. No majority → no flag. It
converges toward existing standards rather than inventing new ones.

---

## Convention checks

Each check is scoped to the file extensions where its regex is meaningful.
All checks can be disabled per-repo via `consistency.json`.

| Check | What it detects | Extensions |
|---|---|---|
| `error_handling_style` | reraise vs swallow vs log_and_continue | .py .js .ts .tsx .jsx |
| `logging_call` | logger vs logging vs print vs console.\* | .py .js .ts .tsx .jsx |
| `naming_convention` | snake_case vs camelCase vs PascalCase | .py .js .ts .tsx .jsx |
| `docstring_coverage` | documented vs undocumented functions | .py |
| `import_organization` | grouped (isort) vs flat imports | .py .js .ts |
| `type_annotation_coverage` | typed vs untyped function signatures | .py .ts .tsx |
| `return_consistency` | explicit_return vs implicit_return | .py .js .ts |

Add project-specific checks in the `CHECKS` list in the script — that's
the intended extension point. See `CONTRIBUTING.md` for the pattern.

---

## Output formats

```
--format text               Human-readable. One finding per line.
--format json               Full structured output. Stable schema.
--format sarif              SARIF 2.1.0. GitHub Security tab + most CI.
--format claude-code-stop   JSON decision for Claude Code Stop hook.
--format claude-code-context JSON context for Claude Code PostToolUse hook.
```

**SARIF** is the key addition for enterprise environments: it renders
natively in GitHub's Security tab and is supported by Azure DevOps,
Buildkite, CircleCI, and most other CI platforms. Each finding carries
structured properties (`dominant_value`, `changed_value`, `dominant_ratio`,
`example_files`) for downstream automation.

---

## Config file

Place `consistency.json` at repo root to tune behaviour without touching
adapter scripts. CLI flags override config values. All fields optional.

```json
{
  "majority_threshold": 0.6,
  "similarity_floor":   0.15,
  "family_size":        8,
  "max_changed_files":  5,
  "ignore_dirs":        ["generated", "migrations"],
  "checks": {
    "docstring_coverage":   false,
    "import_organization":  true
  },
  "warn_only": false,
  "no_llm":    false
}
```

`CONSISTENCY_MAX_CHANGED_FILES` env var overrides `max_changed_files` —
useful for CI environments where you want a different cap than local runs.

---

## LLM enrichment (optional)

Set `ANTHROPIC_API_KEY` (or `CONSISTENCY_LLM_API_KEY`) to enable. The
engine sends the changed file plus the most similar existing files to a
Claude LLM for a sharper natural-language judgment on top of the heuristic
findings. This is a plain HTTP call — not tied to any harness's built-in
subagent feature, so it stays portable. Without a key, heuristic mode
still runs fully.

Override endpoint: `CONSISTENCY_LLM_API_URL`
Override model:    `CONSISTENCY_LLM_MODEL` (default: `claude-sonnet-5`)

---

## Repo layout

```
codebase-consistency-hooks/
├── scripts/
│   └── check_pattern_consistency.py   # the harness-agnostic engine
├── adapters/
│   ├── git/
│   │   ├── pre-commit                 # blocks on commit
│   │   └── pre-push                   # catches --no-verify bypasses at push
│   ├── ci/
│   │   └── github-actions.yml         # PR + push check, with SARIF upload
│   ├── cursor/
│   │   ├── .cursorrules               # legacy Cursor rules file
│   │   ├── consistency-check.mdc      # modern Cursor rules directory format
│   │   └── README.md
│   ├── vscode/
│   │   ├── tasks.json                 # four VS Code tasks
│   │   └── README.md
│   └── codex-cli/
│       └── README.md
├── hooks/
│   └── hooks.json                     # Claude Code: Stop (block) + PostToolUse (context)
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── consistency.json                   # optional per-repo config
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

---

## Setup per harness

### Claude Code

```bash
claude --plugin-dir /path/to/codebase-consistency-hooks
```

Two hooks are active:
- **Stop** (blocking): if changed files diverge from the dominant convention,
  sends Claude back to reconcile before the turn ends.
- **PostToolUse** (non-blocking): injects findings as context after each
  Write/Edit/MultiEdit call — earlier awareness without a hard block.

### Git pre-commit

```bash
cp adapters/git/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

### Git pre-push (catches --no-verify bypasses)

```bash
cp adapters/git/pre-push .git/hooks/pre-push
chmod +x .git/hooks/pre-push
```

### CI (GitHub Actions)

```bash
mkdir -p .github/workflows
cp adapters/ci/github-actions.yml .github/workflows/codebase-consistency.yml
```

Runs on PRs and pushes to main/master. Findings appear in the GitHub
Security tab via SARIF upload — no extra configuration needed.

### Cursor

Modern (rules directory):
```bash
mkdir -p .cursor/rules
cp adapters/cursor/consistency-check.mdc .cursor/rules/consistency-check.mdc
```

Legacy (`.cursorrules`):
```bash
cp adapters/cursor/.cursorrules .cursorrules
```

### VS Code

```bash
mkdir -p .vscode
cp adapters/vscode/tasks.json .vscode/tasks.json
```

Access via Terminal → Run Task. Install the **SARIF Viewer** extension
(`MS-SarifVSCode.sarif-viewer`) to browse SARIF findings inline.

### Codex CLI

See `adapters/codex-cli/README.md` — same JSON-stdin/exit-code family as
Claude Code. Start with `--warn-only` until you've confirmed the blocking
contract against your Codex CLI version.

### Anything else

```bash
python3 scripts/check_pattern_consistency.py --format text   # exit 0 = clean, 1 = findings
python3 scripts/check_pattern_consistency.py --format json   # structured
python3 scripts/check_pattern_consistency.py --format sarif  # SARIF 2.1.0
```

Any harness that can run a shell command and act on the exit code or JSON
output can integrate. No SDK, no framework import, no runtime dependencies.

---

## Run directly

```bash
# Check whatever git says just changed
python3 scripts/check_pattern_consistency.py --format text

# Check specific files
python3 scripts/check_pattern_consistency.py --files path/to/file.py --format text

# Check changes since a ref
python3 scripts/check_pattern_consistency.py --since origin/main --format json

# Lower the majority threshold (flag more aggressively)
python3 scripts/check_pattern_consistency.py --majority-threshold 0.5

# Warn only — never exit 1
python3 scripts/check_pattern_consistency.py --warn-only

# Disable LLM enrichment even if API key is set
python3 scripts/check_pattern_consistency.py --no-llm
```

---

## Known limitations

- The seven built-in checks are regex-based and lean toward Python/JS/TS
  syntax. They're a starting point — extend `CHECKS` for other languages
  or house-specific conventions.
- Structural similarity is a heuristic (normalised-token shingling), not
  full parsing. It catches "same shape, different names" but can miss more
  abstract equivalences. That's what LLM enrichment is for.
- On very large monorepos, the file walk per changed file is uncapped in
  total directories visited (though capped in files matched). Add
  `--ignore-dir` or set `ignore_dirs` in `consistency.json` for noisy
  directories.

---

## Before pushing

Replace `REPLACE_WITH_...` placeholders in:
- `.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `LICENSE`

## Publishing

```bash
git init
git add .
git commit -m "Initial commit: codebase-consistency-hooks v0.3.0"
gh repo create YOUR_USERNAME/codebase-consistency-hooks --public --source=. --remote=origin --push
```

## Giving team access

The engine and adapters are plain files — no bot token, no app to configure.

- **Named collaborators:** `gh repo add-collaborator YOUR_USERNAME/codebase-consistency-hooks their-username --permission read`
- **Public repo:** already set above with `--public`
- **Template repo** (for teams who should fork and adapt): Settings → General → check "Template repository"

## Updating

Anyone using the marketplace adapter:
```
/plugin marketplace update
```

Everyone else (`git`/CI adapters): new behaviour on next `git pull`.
