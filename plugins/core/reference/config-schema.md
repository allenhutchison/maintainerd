# The repo config contract: `.claude/agent-skills.json`

Every agent-skills skill is **repo-agnostic**. The repo-specific details — slug, branch,
language, paths, commands, label names, caps — live in a config file the consuming repo checks
in. This document is the canonical schema. The `bootstrap` skill generates the file; the other
skills read it.

There are two halves to the contract:

1. **`.claude/agent-skills.json`** — structured scalars (this document).
2. **`.claude/guidelines/*.md`** — free-form repo rules the skills read as prose (see
   [Guidelines files](#guidelines-files)).

---

## The "read your repo config" preamble

Every skill in this toolkit starts the same way. When a skill says *"load the repo config (see
config-schema.md)"*, it means exactly this:

1. Read `.claude/agent-skills.json` from the repo root.
2. If it does not exist, **stop** and tell the user:
   > This repo has no `.claude/agent-skills.json`. Run `/bootstrap` to generate it, then re-run me.
   Do not guess values or hardcode another repo's settings.
3. Read only the keys the skill needs. Treat a `null` command as *"this repo has no such step —
   skip it"* (e.g. a Python repo has `commands.build: null`; don't invent a build step).
4. For any key a skill needs that is **absent** (older config, optional section), fall back to the
   documented default below and mention the fallback in your run report so the user can add it.
5. When a skill needs repo-specific *rules* (coding standards, test conventions, invariants),
   read the markdown file named in `guidelines.*` rather than relying on built-in assumptions.

Read with whatever is convenient — the `Read` tool, or `jq` for a single value, e.g.
`jq -r '.defaultBranch' .claude/agent-skills.json`.

---

## Full schema

```jsonc
{
  // ── Identity ────────────────────────────────────────────────────────────
  "repo": "owner/name",            // GitHub slug, e.g. "allenhutchison/pepper". Used in every `gh ... --repo`.
  "defaultBranch": "main",         // Branch PRs target and audits check out. "main" | "master" | ...
  "language": "python",            // "python" | "typescript". Selects audit/test detection rules.

  // ── Commands ────────────────────────────────────────────────────────────
  // The shell commands a skill runs for pre-flight gates. null = step not applicable; skip it.
  "commands": {
    "format":   "uv run ruff format --check",   // formatting check (non-mutating)
    "lint":     "uv run ruff check",            // linter; null if folded into format
    "build":    null,                           // e.g. "npm run build"; null for interpreted repos
    "test":     "uv run pytest",                // the test suite
    "coverage": "uv run pytest --cov --cov-report=json", // optional; used by audit-tests
    "typecheck": null                           // e.g. "npm run typecheck:test"; null if covered elsewhere
  },

  // ── Paths ───────────────────────────────────────────────────────────────
  "paths": {
    "source":       "src/pepper/",              // root of source the audits sweep
    "tests":        "tests/",                    // root of the test tree
    "designDocs":   ["planning/"],               // design/architecture docs (audit-design-docs)
    "productDocs":  ["docs/", "README.md", "AGENTS.md"], // user-facing docs (audit-product-docs)
    "changelogDir": "planning/changelog/",       // where daily-changelog writes YYYY-MM-DD.md
    "researchRadarDir": "planning/research-radar/", // where research-radar writes its digests
    "prTemplate":   ".github/PULL_REQUEST_TEMPLATE.md", // PR body template create-pr enforces
    "skillsDir":    ".claude/skills/"            // local skill scaffolding dir; pre-flight clean-tree exception
  },

  // ── Guidelines (free-form rule files; see below) ─────────────────────────
  "guidelines": {
    "coding":     ".claude/guidelines/coding.md",
    "testing":    ".claude/guidelines/testing.md",
    "invariants": ".claude/guidelines/invariants.md"
  },

  // ── GitHub labels ────────────────────────────────────────────────────────
  "labels": {
    "architecture": "architecture",  // applied to audit-architecture PRs/issues
    "testQuality":   "test-quality", // applied to audit-tests PRs/issues
    "automated":     "automated"     // applied to anything a skill opens unattended
  },

  // ── Per-run caps for the audits (keep review load sustainable) ───────────
  "audits": {
    "prCap":      3,   // audit-architecture: max PRs opened per run
    "issueCap":   5,   // audit-architecture: max issues filed per run
    "testPrCap":  2,   // audit-tests: max PRs per run
    "testIssueCap": 2  // audit-tests: max issues per run
  },

  // ── daily-update meta-skill ──────────────────────────────────────────────
  "dailyUpdate": {
    "subSkills":    ["daily-changelog", "audit-product-docs"], // roster; ONLY list installed skills
    "branchPrefix": "daily-update-",
    "commitSubject": "chore(daily): {date} daily update"       // {date} -> YYYY-MM-DD
  },

  // ── auto-dev pipeline ─────────────────────────────────────────────────────
  "autoDev": {
    "enabled":      true,
    "branchPrefix": "auto/issue-",
    "marker":       "<!-- auto-dev -->",  // HTML comment stamped on bot comments so the pipeline recognizes its own
    "stateLabels": {
      "needsInfo":  "auto:needs-info",
      "planned":    "auto:planned",
      "ready":      "auto:ready",
      "inProgress": "auto:in-progress",
      "parked":     "auto:parked",
      "skip":       "auto:skip"
    },
    "excludedLabels": ["epic", "question", "wontfix", "duplicate", "invalid"], // never auto-build these
    "openPrsAsDraft": true
  },

  // ── research-radar ─────────────────────────────────────────────────────────
  "researchRadar": {
    "themes":    "derive",  // "derive" (infer from repo) | ["explicit", "theme", "list"]
    "userAgent": "research-radar/1.0 (mailto:you@example.com)" // arXiv API courtesy UA
  }
}
```

---

## Field reference

### Identity

| Key | Meaning | Example |
| --- | --- | --- |
| `repo` | GitHub `owner/name`; passed to every `gh --repo`. | `allenhutchison/pepper` |
| `defaultBranch` | Branch PRs target; audits `git checkout` it before sweeping. | `main`, `master` |
| `language` | Selects the language-specific detection rules in `audit-architecture` and `audit-tests`. | `python`, `typescript` |

### `commands`

Each value is a shell command or `null`. A skill that needs a step whose command is `null`
**skips that step** (and says so). Skills must never substitute a different repo's command.

| Key | Used by | Notes |
| --- | --- | --- |
| `format` | create-pr, audits | Non-mutating format check. |
| `lint` | create-pr, audits | May be `null` if the formatter also lints. |
| `build` | create-pr, auto-dev | `null` for interpreted languages with no build. |
| `test` | create-pr, audit-tests, auto-dev | The suite that gates PRs. |
| `coverage` | audit-tests | Optional; emits machine-readable coverage. |
| `typecheck` | create-pr, auto-dev | Optional separate type-check pass. |

### `paths`

All paths are repo-root-relative. `designDocs` and `productDocs` are **arrays** (a repo can have
several doc roots; entries may be directories or individual files). `skillsDir` names the directory
where untracked skill scaffolding may legitimately appear — the audits treat untracked files there
as benign rather than "a human is mid-work."

### `guidelines`

Pointers to the markdown rule files. See [Guidelines files](#guidelines-files).

### `labels`, `audits`, `dailyUpdate`, `autoDev`, `researchRadar`

- `labels.*` — GitHub label names the skills apply. They must already exist in the repo (bootstrap
  offers to create them).
- `audits.*` — per-run caps. Defaults `3/5` (architecture) and `2/2` (test) if absent.
- `dailyUpdate.subSkills` — the ordered roster `daily-update` runs. **Only list skills that are
  actually installed in this repo**, or `daily-update` will try to invoke a missing skill.
  `{date}` in `commitSubject` is replaced with the ISO date.
- `autoDev.*` — the `auto:*` state-machine label names, the bot comment `marker`, the branch
  prefix, the labels that exclude an issue from auto-building, and whether PRs open as drafts.
- `researchRadar.themes` — `"derive"` to infer themes from the repo, or an explicit string array.

---

## Guidelines files

Long-form, judgment-heavy rules don't belong in JSON. They live in markdown files named by
`guidelines.*` and are read by the skills as prose.

| File | Read by | Holds |
| --- | --- | --- |
| `coding.md` | `code-review`, `audit-architecture` | The coding standards a senior reviewer enforces here — error-handling conventions, logging (`logger` vs `print`/`console`), typing expectations, naming, banned patterns. |
| `testing.md` | `audit-tests`, `code-review` | Test conventions — the DB/fixture pattern, what may and may not be mocked, isolation rules, assertion expectations, naming. |
| `invariants.md` | `audit-architecture` | The **load-bearing, repo-specific invariants** that CI doesn't catch. This is the file that replaces the hardcoded repo-specific prose the old per-repo audits carried (e.g. pepper's `app.state` lifespan wiring + `SecretStr` rule; an Obsidian plugin's "use `plugin.logger`, never `console`" + "don't touch the generated-help file"). The audit reads each rule here and checks the diff against it. |

A skill that finds a guidelines file missing or still full of `TODO` markers should note that in
its report (the rule coverage is only as good as these files) and proceed with the
language-generic checks it can still perform.

---

## Worked examples

A complete pepper (`python`) and obsidian-gemini (`typescript`) config live alongside this file as
[`example-pepper.json`](example-pepper.json) and [`example-obsidian.json`](example-obsidian.json).
They double as the round-trip fixtures the verification step checks the schema against.
