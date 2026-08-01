# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Claude Code plugin** — pure markdown, no build system, no runtime. Everything ships as `SKILL.md` files loaded by Claude Code at invoke time. There is nothing to compile, test, or lint. "Shipping" means editing markdown, bumping versions in two JSON files, and writing a CHANGELOG entry.

The plugin name is `oss-contribute`. It exposes five user-invocable skills under that namespace: `find-issues`, `contribute-upstream`, `follow-up`, `log`, and `profile`.

## Repository layout (the parts that matter)

- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` — plugin manifest + marketplace entry. **Both contain a `version` field that must stay in sync** when releasing.
- `skills/<name>/SKILL.md` — one skill per directory. The YAML frontmatter (`name`, `description`, `trigger`) is what Claude Code reads to register the skill; the body is what gets injected into context when the skill is invoked.
- `skills/contribute-upstream/references/*.md` — auxiliary files loaded on-demand by the skill, **not** on-invoke. This split exists to keep the on-invoke token cost down (see "Token budget" below).
- `docs/profile.example.md` — example of the shared user profile that all skills read from.
- `CHANGELOG.md` — Keep-a-Changelog format, semver. Every behavioural change to a skill gets an entry.

There is no `package.json`, no `node_modules`, no test runner. Don't add one without explicit reason.

## How the skills relate

All skills read from one shared profile file resolved in this order (first existing wins): `$CLAUDE_PLUGIN_DATA/profile.md` → `~/.claude/plugins/data/oss-contribute/profile.md` → `~/.claude/skills/oss-contribute/profile.md`. The profile is the single source of truth for stable preferences (watched repos, languages, default GitHub account, default budget). Per-invocation args override the profile but never mutate it.

- `find-issues` is **read-only**, proactive. Searches watched repos and ranks candidates. Hands off to `contribute-upstream` when the user picks one — never auto-invokes.
- `contribute-upstream` is the heavy one — 8 phases from "I hit a bug in this package" to a merged PR, plus escape hatches (issue-only, propose-comment) and a Phase 8 review-response loop.
- `follow-up` is **read-only** through its report phase. Triages everything the user has open upstream (authored PRs + issues) into ball-with-you vs ball-with-them, then hands the actionable ones to `contribute-upstream` Phase 8. Writes (nudge comment, close comment) are per-item and confirmation-gated.
- `log` renders merged PRs into a portfolio artifact. Read-only; local file or stdout, never publishes.
- `profile` is just the view/edit surface for the shared file.

Procedures duplicated across skills have **one authoritative home** (usually inside `contribute-upstream`) and the other skill defers via a one-line pointer. When editing a procedure, find the canonical version first — don't fork.

## Token budget for skills

`contribute-upstream` is large by skill standards (~7.2k on-invoke as of v0.6.0; comparable plugins top out around 3.9k). The `references/` extraction in v0.6.0 was a deliberate refactor to keep on-invoke cost down by deferring content that's only needed in specific phases:

- Procedural gates (BLOCKING criteria, drop conditions, cheap-check commands) stay **inline** in `SKILL.md` — the model needs them to know *what to check*.
- Narrative "why this gate exists" (failure-case stories, full checklists, full Phase 8 procedure) lives in `references/` — loaded only when the model recognises a similar shape or enters that phase.

When adding to `contribute-upstream`, think about whether the addition is procedural (inline) or narrative/conditional (reference file). One-line motivating-case pointers in the inline gates point at `references/case-studies.md#<anchor>` — keep that pattern.

## Conventions baked into the skills (do not weaken)

These are hard rules the skills enforce on behalf of users. If you're editing a skill, do not relax them without an explicit user request and a CHANGELOG entry:

- **No AI-attribution trailers on any commit.** No `Co-Authored-By: Claude`, no `Generated-By`, no "AI assisted" markers — anywhere, including commit bodies and PR bodies. The OSS contribution must read as sole-authored by the GitHub user. This applies to commits *in this repo* too.
- **Never push to an upstream's main repo.** Always work from the user's fork.
- **Pre-PR confirmation gate is BLOCKING.** `gh pr create` never runs without the user seeing the full body + commits + files and explicitly saying yes.
- **Invitation-only repos are a HARD STOP.** Detection happens in Phase 1; do not proceed to Phase 2.
- **Surface, don't suppress.** If a contribution probably won't be accepted (dormant repo, scope mismatch, CLA blocker, duplicate PR, re-implementation disguised as bug fix), the skill says so before doing the work.

## Editing workflow

1. Edit the relevant `SKILL.md` or `references/*.md`. Match the existing voice (terse, second-person imperative; "Do X" not "You should consider doing X").
2. If you change behaviour: bump `version` in **both** `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`, and add a `CHANGELOG.md` entry under a new version header (Keep-a-Changelog format, semver). Look at recent entries for tone — they explain the motivating case and tradeoff, not just the diff.
3. There is no test suite to run. Verification is by re-reading the changed phase end-to-end and (when possible) trying the skill via `/oss-contribute:<skill>` in a Claude Code session with the plugin installed.

## Commit/PR style for this repo

Conventional Commits (`feat:`, `fix:`, `refactor:`, `docs:`, `chore:`). Subject lines describe the user-visible change in this plugin, not the internal mechanism. Recent log is the reference. **Do not add Claude / AI co-author trailers to commits in this repo** — it's the plugin that enforces that rule everywhere else, and it would be embarrassing to violate it here.
