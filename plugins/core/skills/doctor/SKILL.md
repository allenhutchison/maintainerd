---
name: doctor
description: Validate a repo's Maintainerd setup and report what's wrong — the companion to `bootstrap`. Checks that `.claude/maintainerd.json` exists, parses, and conforms to the schema; that the configured paths, commands, and guidelines files resolve; that the GitHub labels the skills apply actually exist; that the daily-update roster only names installed skills; that the auto-dev state labels exist when the pipeline is enabled; and that release config is coherent. Read-only diagnosis by default, grouped PASS/WARN/FAIL with a concrete fix for each finding; offers to create missing labels and points at `/bootstrap` or the guidelines files for the rest. Use when the user asks to "run doctor", "check the maintainerd setup", "validate the config", "why isn't <skill> working", "diagnose the agent-skills config", or after onboarding a repo to confirm it's wired correctly.
---

# Diagnose a repo's Maintainerd setup

This skill is the companion to `bootstrap`: where `bootstrap` **creates** the config contract,
`doctor` **validates** it and everything it points at. It's what you run when a skill misbehaves
("audit-architecture says no config", "daily-update tried to run a skill that isn't installed",
"my audit PRs have no label") or right after onboarding a repo to confirm it's wired correctly.

**It is read-only by default.** It diagnoses and reports; it does not rewrite your config or
guidelines. The one mutation it will *offer* (with confirmation) is creating missing GitHub labels,
since that's the same safe action `bootstrap` offers. Everything else it routes: "run `/bootstrap`
to regenerate X", "fill in `invariants.md`", "fix this key by hand".

## Inputs

- `/doctor` — full read-only check, print the report.
- `/doctor --fix` — additionally offer to create any missing GitHub labels (still asks first).
- `/doctor --run` — additionally *execute* `config.commands.*` to confirm they work (slower, has
  side effects: runs the test/build). Default is the static check (the command's script is defined),
  not running it.

## The check

Read `.claude/maintainerd.json` and the schema (`../../reference/config-schema.md`) — the schema is
the source of truth for what's valid. Then run every check below, collecting findings as
**PASS / WARN / FAIL**:

- **FAIL** — a skill *will* break: no config, invalid JSON, a missing required key, a label a skill
  applies that doesn't exist, a daily-update roster entry that isn't installed.
- **WARN** — degraded but not broken: a path that doesn't resolve, a command whose script is
  undefined, a stubbed `invariants.md`, an unverifiable schedule, an unknown (likely typo'd) key.
- **PASS** — fine; list briefly, don't pad.

### 1. Config presence and validity
- `.claude/maintainerd.json` exists. If not → **FAIL**: "Run `/bootstrap`." Stop here; nothing else
  can be checked.
- It parses as JSON (`jq . .claude/maintainerd.json`). If not → **FAIL** with the parse error and the
  offending line.

### 2. Schema conformance
- Required top-level keys present: `repo`, `defaultBranch`, `language`, `commands`, `paths`,
  `guidelines`. Missing → **FAIL**.
- `language` is `python` | `typescript` (else **WARN**: audits fall back to language-generic checks).
- Types match the schema (arrays are arrays, caps are numbers, `commands.*` are strings or `null`).
- **Unknown keys** not in the schema → **WARN** (usually a typo, e.g. `commands.tests` for `test`;
  the skill reading it will silently miss the value).

### 3. Paths resolve
For each `config.paths.*`: the directory/file exists in the repo. `designDocs`/`productDocs` are
arrays — check each entry. A configured `prTemplate` that doesn't exist → **WARN**. `source`/`tests`
not existing → **FAIL** (the audits sweep nothing). `skillsDir` missing is fine (it may not exist yet).

### 4. Commands
For each non-`null` `config.commands.*`: confirm it's plausibly runnable.
- **TypeScript**: an `npm run <x>` command → `<x>` exists in `package.json` `scripts`. Missing → **FAIL**.
- **Python**: the tool (`ruff`, `pytest`, …) is on `PATH` or declared in `pyproject.toml` dev deps.
- With `--run`: actually execute each and report pass/fail (this is the real proof, but slow).

### 5. Guidelines health
- Each file named in `config.guidelines.*` exists. Missing → **WARN** (the skill that reads it falls
  back to generic checks).
- `invariants.md` is more than a stub — flag if it's empty or still all `TODO`/template
  (**WARN**: "audit-architecture is only as good as the invariants listed here").
- `guidelines.release` present iff `config.release` is non-null (a versioned-release repo should have
  release gates documented; a `release: null` repo shouldn't need the file).

### 6. GitHub labels exist
```bash
gh label list --repo <config.repo> --json name --jq '.[].name'
```
Every label in `config.labels.*` must exist. Missing → **FAIL** (the skill's `gh ... --label` call
errors at runtime). With `--fix`, offer to create the missing ones (same `gh label create` as
`bootstrap`). If `config.autoDev.enabled`, the `config.autoDev.stateLabels.*` must also exist.

### 7. daily-update roster
Every skill in `config.dailyUpdate.subSkills` must be installed/available (a Maintainerd skill name,
or a repo-local skill that exists). A roster entry that resolves to nothing → **FAIL** ("daily-update
will try to invoke a missing skill"). Cross-check against the plugins actually installed.

### 8. auto-dev coherence (only if `config.autoDev.enabled`)
- All six `stateLabels.*` exist on GitHub (covered in check 6).
- `marker` is a non-empty HTML comment; `branchPrefix` is set; `excludedLabels` is an array.
- If `autoDev.enabled` is `false`, skip — note it as a PASS ("auto-dev disabled").

### 9. release coherence
- `config.release` is `null` → PASS ("continuous-deploy repo, no versioned releases").
- Non-null → `notesFile` (if set) exists; `versionCommand` references a real mechanism (an
  `npm version`/script that exists, a tool on PATH); `versionPushesTag` is a boolean. Gaps → **WARN**.

### 10. Schedules (best-effort, advisory)
Scheduled runs live in Claude Code's scheduling config, not the repo, so this can't always be
verified from here. Don't fail on it. Advise: the audits and `daily-update` are designed to run on a
schedule — list which skills the repo has installed that *want* scheduling, and suggest the user
confirm they're wired via the `schedule` skill. Mark **WARN** only if you can positively tell an
expected schedule is absent.

## Report

```text
maintainerd doctor — <repo>   (<config.language>, default branch <config.defaultBranch>)

FAIL (<n>) — skills will break until fixed:
  - labels: `security` missing on GitHub — audit-security's PRs will error.   Fix: /doctor --fix  (or gh label create security)
  - daily-update roster lists `triage-issues`, which isn't installed.          Fix: install it, or drop it from config.dailyUpdate.subSkills

WARN (<n>) — degraded:
  - guidelines/invariants.md is still the bootstrap stub.                       Fix: fill in the repo's load-bearing invariants
  - paths.prTemplate (.github/PULL_REQUEST_TEMPLATE.md) doesn't exist.          Fix: create it, or set the key to null

PASS (<n>): config parses · schema OK · source/tests resolve · commands defined · 4/5 labels present · release: null (continuous-deploy)

Summary: <n> FAIL, <n> WARN — fix the FAILs before relying on the affected skills.
```

If everything passes: `maintainerd doctor — all green. <n> checks passed.` Don't pad a clean run.

## What not to do

- **Don't rewrite the config or guidelines.** doctor diagnoses; `bootstrap` (or the user) fixes.
  The only mutation it performs is creating missing labels, and only with `--fix` + confirmation.
- **Don't run `config.commands.*` without `--run`.** They have side effects (and can be slow); the
  default check is static.
- **Don't fail on things it can't verify** (schedules) — mark advisory, not FAIL.
- **Don't report PASS for a check it skipped** — if a tool needed to verify something is missing, say
  "couldn't verify", not "OK".
- **Don't create labels silently** — always confirm, like `bootstrap`.
