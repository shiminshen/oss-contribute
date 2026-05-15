# Changelog

All notable changes to this plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned

- `do` / `guide` / `adaptive` operating modes (per-session, persisted)
- Per-repo conventions cache to speed up repeat contributions

## [0.7.1] — 2026-05-15

### Fixed — `log` skill bugs found by first real run

Initial v0.7.0 of `log` was written from spec without dry-running against a live `gh` install. Running it against the first two real merged PRs (`cloudflare/workers-sdk#13908`, `santifer/career-ops#600`) surfaced multiple bugs in the same session:

- **`gh search prs --state merged` is invalid.** The CLI accepts `open|closed` only. Correct form: `--merged` (flag) + `--merged-at ">=DATE"`. Phase 2 rewritten with the correct invocation. The same bug existed in the README's "Checking your pipeline" one-liner — fixed there too.
- **`additions`, `deletions`, `changedFiles`, `mergedAt` are not available on `gh search prs`.** They only come back from `gh pr view <url>`. Phase 2 redesigned as two-pass: search returns URLs, then per-PR view fetches diff stats and merged timestamp. N+1 round-trips is fine for a bounded set (<30 PRs in a typical window).
- **PR bodies can contain raw control characters** that break multi-record `jq` parses. Body is now fetched separately per PR via `gh pr view <url> --json body --jq .body` rather than included in the search response.
- **"First non-empty paragraph" was too naive.** PRs commonly lead with `Fixes #N` or `Closes #N` reference-only paragraphs that aren't narrative. Body extraction now explicitly skips leading reference-only paragraphs (`Fixes`, `Closes`, `Resolves`, `Related:`, `Refs`, optionally `<owner>/<repo>`-prefixed) before picking the first informative paragraph. Also added `## Summary by CodeRabbit` and similar bot summaries to the strip list.
- **README Receipts** missing `santifer/career-ops#600`. Added.

The receipt-shape pattern matters more than the spec discipline here: writing a skill against documented CLI flags is no substitute for actually invoking the CLI on a real account. This is exactly the failure mode the plugin's own `contribute-upstream` Phase 3 step 0 catches in upstream PRs — applied to ourselves now.

## [0.7.0] — 2026-05-15

### Added — `log` skill

Ports the canonical Phase B procedure from the personal `career-ops` umbrella into the plugin as a first-class skill. Generates a portfolio artifact from your merged upstream PRs in a configurable window (default 90 days).

Output is hybrid by user preference: a table-of-contents table at the top (date, repo, PR#, diff size) so a reader can skim, plus one detail block per PR below (title, merge date, diff, first non-empty paragraph of the body verbatim, link). Sources content from `gh` + the PR title/body only — no inferred narrative, no diff-stat fabrication.

Hard rules baked in, matching the rest of the plugin:

- **No auto-publishing.** Writes to `~/Documents/oss-contribute/log-<YYYY-MM-DD>.md` or stdout, never to LinkedIn / X / a Gist / the consumer repo.
- **Ask every time when accounts are ambiguous.** Same rule the other skills enforce.
- **Restore the active gh account** after the skill switches to the profile-default for the run.
- **No PR-body fabrication.** Empty/templated body → fall back to the title alone, do not write inferred context.
- **Read-only.** Never modifies the profile, never opens a PR.

Out of scope (deliberately — same reasoning as the rejected `pipeline` skill):

- Open-PR status (`gh search prs --author @me --state open` is one line, see README "Checking your pipeline")
- Journal of *bailed* contribution attempts (one-liner in the consumer repo's `CONTRIBUTIONS.md` is enough; not skill-shaped)

The `career-ops` umbrella's Phase B will be updated in a follow-up to defer to this skill so there is one canonical home, per the plugin's "one authoritative home per procedure" rule.

## [0.6.1] — 2026-05-15

### Changed — listing-readiness pass (docs + metadata)

Pre-submission cleanup for the Anthropic plugin marketplace. No behaviour change.

- Trimmed `plugin.json` `description` from 247 → 108 chars so listing cards don't truncate.
- Added `CONTRIBUTING.md` — extracts editing workflow, token-budget tiering, hard rules, and PR checklist from `CLAUDE.md` so external contributors have an on-ramp without reading internal-conventions files.
- Added `CLAUDE.md` to version control (was untracked; referenced from CONTRIBUTING.md).
- README: moved the "Why this instead of [other plugin]" comparison below the demo flows. New entrants benefit from in-README comparisons (Vite, Bun, pnpm pattern), but leading with one makes the page open defensively. Compromise: keep it, demote it.
- README receipts: swapped placeholder for first real merged PR — [`cloudflare/workers-sdk#13908`](https://github.com/cloudflare/workers-sdk/pull/13908) (+14/-1).

## [0.6.0] — 2026-05-14

### Changed — `contribute-upstream` reference-file refactor

`contribute-upstream`'s on-invoke token cost was ~9.4k — the largest skill in the everything-claude-code 89-skill plugin tops out at ~3.9k. The bulk came from three categories of content that don't all need to be loaded every invocation: the full Phase 8 review-response procedure (only relevant when a PR is open), the seven-dimension convention checklist (used in two phases), and the ~8 documented failure-case narratives (each tied to one specific gate).

Extracted those into `skills/contribute-upstream/references/`:

- `references/convention-checklist.md` — single source for the Phase 3 step 0 pre-coding scan + Phase 4 audit. Phase 3 and Phase 4 in SKILL.md now load this on demand.
- `references/phase-8-review.md` — full 7-step procedure + Phase 8 hard rules. Phase 8 in SKILL.md collapses to trigger condition + load instruction.
- `references/case-studies.md` — eight documented failure cases consolidated (`mastra#16422` late assignee, `vercel/ai#13962` dead-area, `assistant-ui#4009` / `mastra#16383` / `drizzle-orm#5755` already-fixed-on-main, `openai/codex` invitation-only, `%5F` literal-token search, `drizzle-orm#5755` feature-gone, `CopilotKit#4798` convention divergence, `mastra#16514` late dup-PR). Each gate inline in SKILL.md keeps a one-line motivating-case pointer.

Result: contribute-upstream on-invoke dropped from **~9.4k → ~7.2k** (~24%). Still above the popular-plugin median but now in the ballpark of the largest comparable skills (springboot-patterns at 3.9k). Behaviour unchanged.

The procedural gates (BLOCKING criteria, drop conditions, cheap-check commands) stay inline so the model doesn't need a reference fetch to know *what to check*. The narrative *why* — the motivating case story — moves to references/ where it loads only when the model wants to recognise a similar shape.

## [0.5.0] — 2026-05-14

### Added — `contribute-upstream` Phase 3 step 0: Convention scan (BLOCKING before any code)

Catch style divergence at **prevention** time, not just at audit time. The existing Phase 4 Convention audit catches divergence right before commit — by then the code is already written and any fix means churn. The new Phase 3 step 0 forces the model to read the surrounding code and absorb its conventions before writing the failing test or the fix.

Two-stage check:

- **Phase 3 step 0** — pre-coding scan. Open the closest existing test file end-to-end; note test structure, helper/mock-factory patterns, naming conventions, cross-cutting setup placement, comment density, lifecycle/async idioms, import style. Read recent merged PRs touching adjacent files (`gh pr list --search "<path>" --state merged --limit 3`) — they show what conventions the maintainers *enforce in review*, which may differ from what older files still show.
- **Phase 4 Convention audit** — post-write verification. Re-read your changes against the same checklist; catch what slipped through. Updated wording to make the prevention/verification pairing explicit.

Lesson from CopilotKit#4798 stands: the original failure produced review churn because three style divergences shipped in the first commit (ad-hoc inline cast, fetch stub repeated per test instead of `beforeEach`, raw Map exposed instead of factory controller functions). The pre-coding scan would have flagged all three before they were written.

## [0.4.1] — 2026-05-14

### Changed — compress duplicated passages in `find-issues`

Each duplicated procedure now has one authoritative home (in `contribute-upstream`) and `find-issues` defers to it with a short pointer. Saves ~15 lines, no behavior change.

- **Scope/intent ambiguity drop criterion** — full criteria + the `drizzle-orm#5755` documented case live in `contribute-upstream` Phase 3 step 4. `find-issues` Phase 3 keeps a one-line drop criterion + pointer.
- **Invitation-only upstream drop criterion** — full check + the `openai/codex` documented case live in `contribute-upstream` Phase 1 step 2. `find-issues` Phase 3 keeps a one-line drop criterion + pointer.
- **Phase 4 pre-output freshness re-check** — now refers to "the Phase 3 duplicate-PR search" instead of re-listing the token types. The procedure stays in Phase 3 as the single source of truth.

## [0.4.0] — 2026-05-13

### Performance — search speedups

The bottleneck users feel during `find-issues` and `contribute-upstream` Phase 1 is the search step: many sequential `gh` round-trips. This release cuts API round-trips and replaces sequential dispatch with explicit-parallel dispatch.

- **Single OR-query for labels** (both skills). `gh search issues --label` is AND, not OR — running three separate searches for `good first issue` / `help wanted` / `bug` is 3× the round-trips. Replaced with one `--query 'repo:... state:open (label:"good first issue" OR label:"help wanted" OR label:"bug")'` call.
- **GraphQL batch enrichment in `find-issues` Phase 3.** Replace N `gh issue view --json` round-trips with a single `gh api graphql` call that returns assignees + labels + comments + `closedByPullRequestsReferences` for all candidates at once. ~30× round-trip reduction at typical pool sizes. Fallback to batched-parallel `gh issue view` if GraphQL is unavailable.
- **True-parallel Bash dispatch — explicit instructions in both skills.** Previous text said "in parallel" but the model often serialised calls because instructions read sequentially. Now: "Send ONE message containing one Bash tool call per repo/token. Do not run them sequentially." Applies to per-repo discovery in `find-issues` Phase 2 and the 4 token queries in both skills' duplicate-PR searches.

### Added — `find-issues` Phase 3 repo-activity scoring

Issues in repos that don't merge anything are dead ends; issues in repos that merge weekly are bets worth making. Per watched repo, fetch one extra parallel signal (recent merge count via `gh search prs --state merged --merged ">=30d"`) and classify into:

- **Hot** — ≥1 merge in last 7d + recent push: strong boost
- **Active** — ≥1 merge in last 30d: normal weight
- **Slow** — 0 merges in 30d: demote one tier
- **Dormant** — no merges + no default-branch commits in 60d: drop, surface to user so they can prune the watchlist

Stars are tiebreaker only — never a primary score.

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
