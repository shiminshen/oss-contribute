# Changelog

All notable changes to this plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned

- `pipeline` skill — status of your open/merged PRs across forks
- `log` skill — generate a portfolio entry from merged contributions
- `do` / `guide` / `adaptive` operating modes (per-session, persisted)
- Per-repo conventions cache to speed up repeat contributions

## [0.3.0] — 2026-05-13

### Added — features ported from competitor scans

- **`contribute-upstream` Phase 6b — Propose action.** When an upstream issue exists but the fix is too large or design-sensitive for one session, post a structured proposal comment (problem restatement, root-cause analysis, design sketch, open questions, what-I'd-need-to-ship-it) on the existing issue. Pattern adapted from `mainnebula/token-steward`'s Fix/Review/Propose action types. Fills the gap between "open issue" and "open PR" — gauges maintainer interest before anyone writes code.
- **`contribute-upstream` Phase 8 — Respond to PR review.** Round-trip handling between PR open and merge: fetch review state, classify each feedback item (apply / push back / clarify / out-of-scope), summarise the plan to the user, apply only after confirmation, reply on every thread, never silently re-push. Pattern adapted from `LuciferDono/contribute`'s PR-review phase, with stricter no-force-push / no-rebase-mid-review rules.

### Changed — hardened OSS-contribution norms

- **No AI-attribution trailers on commits.** Explicit hard rule banning `Co-Authored-By: Claude`, `Generated-By`, or any "AI assisted" marker on commits, in commit-message bodies, or in PR bodies. OSS maintainers treat these as noise. Identity comes from `gh api user --jq '.login'` and `git config user.name` / `user.email`. Pattern adapted from `LuciferDono/contribute` Rule 2.

## [0.2.0] — 2026-05-13

### Added — pre-clone / pre-PR safety gates

Three rounds of preflight hardening, each grounded in a documented failure case from the session that produced them:

- **`find-issues` Phase 3 — scope/intent ambiguity drop criterion.** For "X is missing/broken in vN.0.0-beta/rc.M" bugs against packages mid-rewrite, briefly verify X exists in the new code paths before classifying as ripe. If wholly absent, the "fix" is a re-implementation, not a bug fix — demote to issue-comment-only. Lesson from `drizzle-team/drizzle-orm#5755` where the v1 rewrite dropped the entire `casing` runtime pipeline; adding the type alone would have shipped a misleading API.
- **`find-issues` Phase 3 — invitation-only upstream drop criterion.** If `docs/contributing.md`, `CONTRIBUTING.md`, or `.github/pull_request_template.md` contains "invitation only" / "do not accept unsolicited" / "closed without review", drop the candidate. Lesson from `openai/codex` closing uninvited PRs unread.
- **`contribute-upstream` Phase 3 — tractability gate (BLOCKING).** Before writing the fix: catch "feature gone, not bug" (re-implementation framed as bug fix) and "half-fix risk" (type-only patches that suppress TS errors while leaving runtime broken). Switch to Phase 6 (issue-only) when either applies.

### Changed

- **`contribute-upstream` Phase 1 step 2** now expands documentation lookups to include `docs/CONTRIBUTING.md`, `docs/contributing.md`, lowercase `.github/pull_request_template.md`, etc., and treats the first hit as authoritative. Many projects put the real policy at `docs/contributing.md` while the root file is missing or a stub.
- **`contribute-upstream` Phase 2 step 0 (new)** — re-verify the duplicate-PR search immediately before clone. Catches PRs landed in the hunt → contribute window; saves 15–25 min of wasted clone time. Lesson from `vercel/next.js#93700` ↔ `#93725`.
- **Both skills now strongly recommend `general-purpose` subagent dispatch** for context-heavy phases (per-repo discovery in `find-issues` Phase 2; rules-of-the-road preflight in `contribute-upstream` Phase 1 step 2). Keeps raw JSON / doc content out of the main agent's context.
- **Both skills' duplicate-PR search is now token-based** (issue number, URL-encoded literals, backticked identifiers, error fragments) rather than title-keyword. Title keywords miss PRs whose title describes the implementation. Lesson from `vercel/next.js#93700` where the closing PR `#93725` was findable only via the `%5F` literal token.

## [0.1.0] — 2026-05-13

### Added

- `find-issues` skill — ranked shortlist of ripe issues across watched repos
- `contribute-upstream` skill — reactive flow for bugs hit in third-party deps: pre-flight gate, repro bridge, fix, PR with confirmation gate, local-patch handoff
- `profile` skill — shared preferences for watched repos, languages, default GitHub account, budget, and "ripe" heuristic
- `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` so the repo doubles as its own marketplace
