# Maintainerd — skill roadmap

A living catalog of skills for the Maintainerd suite: what's shipped, what's ready to extract
from an existing repo, and what's net-new. Use it to pick the next thing to build and to keep the
suite's scope coherent.

**Status legend**
- ✅ **Shipped** — in the marketplace today.
- ♻️ **Extract** — a working version already lives in `pepper` and/or `obsidian-gemini`; the job is
  the same generalize-against-config pass we've already done (swap repo tokens for `config.*`, lift
  repo specifics into `guidelines/*.md`).
- ✨ **Net-new** — no existing source; design from scratch.

**Design conventions every new skill follows** (so the suite stays consistent):
1. Opens by reading `.claude/maintainerd.json` (the "Load the repo config" preamble); stops and
   points the user to `/bootstrap` if it's missing.
2. Repo-specific *scalars* come from the config; repo-specific *rules* come from `guidelines/*.md`.
3. Anything that opens PRs runs the repo's pre-flight via `create-pr` (or `config.commands.*` inline).
4. Unattended/scheduled skills are **capped per run**, **dedup against open work**, and where
   appropriate are **silent on clean** (no findings → no PR, no issue, no report).
5. Never auto-merges. Human-in-the-loop for anything outward-facing.

---

## Shipped (v0.1)

| Plugin | Skills |
| --- | --- |
| `maintainerd-core` | `bootstrap` |
| `repo-ops` | `create-pr`, `code-review`, `daily-changelog`, `daily-update` |
| `audits` | `audit-architecture`, `audit-tests`, `audit-design-docs`, `audit-product-docs` |
| `research` | `research-radar` |
| `auto-dev` | `auto-dev`, `review-queue` |

---

## repo-ops candidates (the dev/PR lifecycle)

| Skill | Status | Source | What it does |
| --- | --- | --- | --- |
| `release` | ♻️ Extract | `release-process` in both repos | Version bump → promote the daily changelog into versioned release notes → tag → GitHub release → verify. The "ship a version" counterpart to `daily-changelog`'s "record a day." |
| `address-review` | ♻️ Extract | `coderabbit-review` (global) | The review-response loop: fetch every thread (CodeRabbit / Gemini / human) → classify → fix real ones (one commit each) → reply to every thread → wait for the next round. Closes the PR lifecycle alongside `create-pr`. |
| `triage-issues` | ♻️ Extract | `triage-issues` in obsidian-gemini | Label unlabeled issues via `gh`. Conservative — never closes or reassigns, only adds existing labels. Could also live in a dedicated `issues` plugin. |

## audits candidates (capped, silent-on-clean findings)

| Skill | Status | Source | What it does |
| --- | --- | --- | --- |
| `audit-security` | ✨ Net-new | wraps `/security-review` | Dependency CVEs, secret scanning, dangerous patterns (injection, unsafe deserialization, shelling out with user input). The most obviously-missing audit. |
| `audit-deps` | ✨ Net-new | — | Outdated / deprecated / unused dependencies, lockfile drift, license issues. Highly generalizable — the config already names the ecosystem (`language`, commands). |
| `audit-todos` | ✨ Net-new | — | Sweep `TODO`/`FIXME`/`HACK`/`XXX`, age them against blame, and file issues for stale ones. Lightweight, language-agnostic, satisfying. |
| `audit-config` | ✨ Net-new | — | Meta-maintenance of the plumbing: validate CI workflows, `.env.example` ↔ real settings sync, schedule definitions, and the repo's own `maintainerd.json`/guidelines health. |

## research candidates (proactive — surface outside knowledge, open a PR/digest)

| Skill | Status | Source | What it does |
| --- | --- | --- | --- |
| `deep-research` | ♻️ Extract | `deep-research` (global) | The fan-out → fetch → adversarially-verify → cited-report harness. Drops straight into `research`. |
| `prior-art` | ✨ Net-new | — | GitHub/code search for similar implementations, competing libraries, or existing solutions before you build. Natural feeder into `auto-dev`'s planning step. |
| `dependency-watch` | ✨ Net-new | — | Track upstream changelogs and *coming* breaking changes in deps you actually use (distinct from `audit-deps`'s "what's stale right now"). |

## auto-dev candidates

| Skill | Status | Source | What it does |
| --- | --- | --- | --- |
| `plan` | ✨ Net-new (extract from auto-dev) | inline in `auto-dev` | Stand-alone "draft an implementation plan for issue #N for maintainer approval." Pulling it out lets you plan without running a full tick. |

## meta / suite-health

| Skill | Status | Source | What it does |
| --- | --- | --- | --- |
| `doctor` | ✨ Net-new | — | The companion to `bootstrap`: validate `.claude/maintainerd.json` + guidelines health, confirm the configured labels exist on GitHub, confirm schedules are wired, flag stubbed `invariants.md`. The thing you run when a skill misbehaves. |
| `sync-config` | ✨ Net-new | — | Re-run bootstrap's detection when the repo changes (new test command, renamed dirs) and propose a config diff for approval. |
| `skill-author` | ♻️ Extract | `skill-author` in pepper | A *meta* skill that scaffolds a new config-driven Maintainerd skill (frontmatter, config preamble, conventions). On-brand and accelerates everything else in this list. |

---

## New category: journal / worklog (cross-repo, maintainer-scoped)

The existing global **`worklog`** skill summarizes a day's work as a dated entry in an Obsidian
vault — creating a session-summary note, linking it from the project hub note, adding a one-line
entry under the daily note, and scanning the day's merged GitHub PRs for completeness. It's a
strong, heavily-used skill and clearly belongs in the suite.

It doesn't fit the existing plugins cleanly, because unlike every shipped skill it's **not
repo-scoped** — it writes to a *personal vault* and spans projects. That suggests a new category
(working name **`journal`**) for maintainer-scoped narration/record-keeping, alongside future
siblings like a decision-log or a weekly-review skill.

**Design implication worth resolving when we implement it:** worklog needs config the current
contract doesn't carry — a vault path, the project-hub note, the daily-note location. Two options:
- Add a `journal` section to `maintainerd.json` (vault path, hub-note pattern, date format), or
- Keep journal config in a separate `~/.claude/`-level file since it's per-user, not per-repo
  (the vault is the same across all the user's repos).

The per-user-not-per-repo nature is the key question. Leaning toward a user-level config that the
repo-level `maintainerd.json` can point at, so one vault setup serves every onboarded repo.

| Skill | Status | Source | What it does |
| --- | --- | --- | --- |
| `worklog` | ♻️ Extract | `worklog` (global) | Summarize the day's shipped work into the user's Obsidian vault: session-summary note + project-hub link + daily-note line, enriched from merged PRs. |
| `weekly-review` | ✨ Net-new | — | Roll the week's worklogs/PRs into a higher-level review note. (Speculative sibling.) |
| `decision-log` | ✨ Net-new | — | Capture notable technical decisions as dated ADR-style notes. (Speculative sibling.) |

---

## Suggested v0.2 batch

Closes the loop *bootstrap → build → review → ship → keep-healthy*, and four of five have existing
source to extract from:

1. `release` (♻️)
2. `address-review` (♻️)
3. `audit-security` (✨)
4. `audit-deps` (✨)
5. `doctor` (✨)

Plus `worklog` (♻️) as the first skill in the new `journal` category, pending the user-vs-repo
config decision above.
