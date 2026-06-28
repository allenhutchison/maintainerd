---
name: bootstrap
description: Generate the maintainerd config contract for a repo — write `.claude/maintainerd.json` and scaffold `.claude/guidelines/{coding,testing,invariants}.md` so the repo-ops, audits, research, and auto-dev skills can run here. Inspects the repo (language, GitHub slug, default branch, source/test dirs, lint/format/build/test commands), confirms anything ambiguous, and seeds starter guidelines from existing CLAUDE.md/AGENTS.md. Use when the user asks to "bootstrap this repo", "set up maintainerd", "create the maintainerd config", "configure the maintainer skills", "onboard this repo to maintainerd", or when any other maintainerd skill reports the config is missing. Idempotent — re-running re-confirms and only rewrites changed keys; never clobbers hand-edited guideline prose.
---

# Bootstrap a repo for the Maintainerd toolkit

This skill writes the **config contract** that every other Maintainerd skill reads. After it
runs, the repo has:

- `.claude/maintainerd.json` — structured config (schema in
  [`../../reference/config-schema.md`](../../reference/config-schema.md)).
- `.claude/guidelines/coding.md`, `testing.md`, `invariants.md` — free-form rule files the audits
  and code-review read.

The goal is a config that's *correct on the first read* — detect everything you safely can, ask
the user only about what's genuinely ambiguous, and be honest in the final report about what still
needs human judgment (especially `invariants.md`).

Read the schema reference first: [`../../reference/config-schema.md`](../../reference/config-schema.md).
Every key you write must match it.

## Workflow

### 1. Detect whether this is a re-run

```bash
cat .claude/maintainerd.json 2>/dev/null
```

- **Exists** → this is an update. Read it, keep every value the user has set, and only re-confirm
  values that look stale or that the user asks to change. **Never** overwrite `.claude/guidelines/*.md`
  whose body is more than the scaffold (see step 6). Report what changed.
- **Missing** → fresh bootstrap. Continue.

### 2. Detect identity and language

Run these and read the results — don't assume:

```bash
gh repo view --json nameWithOwner -q .nameWithOwner   # -> repo
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's@^origin/@@' \
  || git branch --show-current                         # -> defaultBranch (fallback to current branch)
ls pyproject.toml package.json go.mod Cargo.toml 2>/dev/null
```

- `pyproject.toml` present, no `package.json` → `language: "python"`.
- `package.json` present, no `pyproject.toml` → `language: "typescript"`.
- Both, or neither → **ask** the user which language the skills should target (this drives the
  audit/test detection rules). The toolkit currently ships Python and TypeScript rule sets; if the
  repo is another language, tell the user the audits will fall back to language-generic checks only.

### 3. Detect commands

Read the project manifest and pull the real commands rather than guessing:

- **Python** (`pyproject.toml`): look for ruff (`format` → `… ruff format --check`, `lint` →
  `… ruff check`), the test runner (`pytest` → `… pytest`). Honor the project's runner prefix
  (`uv run`, `poetry run`, `hatch run`, or bare) — check for `uv.lock`/`poetry.lock`. `build` is
  almost always `null`. `typecheck` → mypy/pyright if configured, else `null`.
- **TypeScript** (`package.json` `scripts`): map `format`/`format-check`, `lint`, `build`,
  `test`, `typecheck`/`typecheck:test` to whatever scripts exist. Use the exact `npm run <script>`
  names present; don't invent scripts.
- Show the user the command block you inferred and let them correct it. A wrong test command
  silently breaks create-pr and auto-dev, so confirm this one explicitly.

### 4. Detect paths

```bash
ls -d src/*/ src/ tests/ test/ planning/ docs/ 2>/dev/null
ls README.md AGENTS.md CLAUDE.md .github/PULL_REQUEST_TEMPLATE.md .github/pull_request_template.md 2>/dev/null
```

Infer:
- `source` — the source root. For Python, the single package dir under `src/` (e.g. `src/pepper/`);
  for TS, usually `src/`.
- `tests` — `tests/` or `test/` (Python convention vs TS convention — detect which exists).
- `designDocs` / `productDocs` — `planning/` is usually design; `docs/`, `README.md`, `AGENTS.md`,
  `CLAUDE.md` are usually product/contributor docs. Present your split and let the user adjust.
- `changelogDir`, `researchRadarDir` — default under `planning/` if it exists, else propose a
  location.
- `prTemplate` — whichever template path exists; if none, note it for step 5.
- `skillsDir` — where this repo keeps local skill scaffolding (`.claude/skills/` or `.agents/skills/`).

### 5. Confirm the ambiguous bits, then assemble the JSON

Use `AskUserQuestion` for anything you couldn't determine confidently — typically: which plugins
they installed (so the `dailyUpdate.subSkills` roster lists only real skills), whether auto-dev is
`enabled`, label names if they differ from the defaults, and the research-radar `userAgent` email.

Defaults to apply without asking (state them in the report):
- `labels`: `architecture` / `test-quality` / `security` / `automated`.
- `audits`: `3 / 5 / 2 / 2`.
- `autoDev` label names: the `auto:*` set from the schema.
- `autoDev.excludedLabels`: `["epic", "question", "wontfix", "duplicate", "invalid"]`.

Write `.claude/maintainerd.json` with the full schema. Every command the repo doesn't have must be
explicit `null`, not omitted. Validate it parses:

```bash
jq . .claude/maintainerd.json >/dev/null && echo "config valid"
```

### 6. Scaffold the guidelines files

Create `.claude/guidelines/` and write the three files **only if they don't already exist** (on a
re-run, leave populated ones untouched). Seed each from what the repo already documents — read
`CLAUDE.md` / `AGENTS.md` / any `CONTRIBUTING.md` and lift the relevant rules.

- `coding.md` — coding standards (error handling, logging convention, typing expectations, banned
  patterns like `print`/`console`). Seed from the repo's existing guidance; mark gaps with `TODO`.
- `testing.md` — test conventions (DB/fixture pattern, what may/may not be mocked, isolation,
  assertion expectations). Seed from existing test docs; `TODO` the rest.
- `invariants.md` — the load-bearing, repo-specific invariants the audit-architecture enforces.
  **This is the file that most needs a human.** Seed it with a short template and any invariants you
  can extract from `CLAUDE.md`/`AGENTS.md`, but clearly `TODO`-mark it: the audit is only as good as
  the rules listed here.

Each scaffolded file should start with a one-line header explaining what reads it and that it's
safe to edit freely.

### 7. Offer the PR template

If `paths.prTemplate` points at a file that doesn't exist, offer to create a minimal one
(`## Summary`, `## Changes`, `## Test plan`). Don't force it — some repos don't use templates.

### 8. Offer to create the labels

The labels in the config must exist in GitHub for the audits and auto-dev to apply them. Offer:

```bash
gh label create architecture --color BFD4F2 --description "audit-architecture findings" 2>/dev/null || true
gh label create test-quality --color D4C5F9 --description "audit-tests findings" 2>/dev/null || true
gh label create security     --color D93F0B --description "audit-security findings" 2>/dev/null || true
gh label create automated    --color EDEDED --description "Opened by a Maintainerd skill" 2>/dev/null || true
# auto:* labels only if auto-dev is enabled
```

Ask before creating; don't mutate the repo's label set unprompted.

### 9. Report

Tell the user, concisely:
- The path of the config written and that it validated.
- The detected values for `repo`, `defaultBranch`, `language`, and the command block (the
  high-blast-radius ones).
- Which guideline files were created vs left alone, and that **`invariants.md` needs human review**.
- Whether labels / PR template were created or skipped.
- The reminder to **commit `.claude/maintainerd.json` and `.claude/guidelines/`** so scheduled
  cloud agents pick them up.

## What not to do

- **Don't invent commands or paths.** If you can't detect something and the user doesn't know,
  write `null` (commands) or leave the path out and flag it — a wrong value is worse than an
  absent one the skill can report on.
- **Don't clobber hand-edited guidelines.** On a re-run, a guideline file with real content is the
  user's; only touch it if they explicitly ask.
- **Don't list uninstalled skills in `dailyUpdate.subSkills`.** The roster must match what's
  actually installed, or `daily-update` will try to run a missing skill.
- **Don't create labels or a PR template without asking.** These mutate the repo's GitHub/file
  state; confirm first.
- **Don't commit.** Write the files and let the user review and commit them.
