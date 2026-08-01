# Changelog

All notable changes to this plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned

- `do` / `guide` / `adaptive` operating modes (per-session, persisted)
- Per-repo conventions cache to speed up repeat contributions

## [0.11.0] — 2026-08-01

### Added — `follow-up`, a fifth skill: what of everything I opened is waiting on me?

`/oss-contribute:follow-up` lists every PR and issue you have open upstream and sorts them by who owes the next move — changes requested, required checks red, conflicts, an unanswered maintainer question, a stale-bot deadline, work superseded by someone else's merge — then hands the actionable ones to `contribute-upstream` Phase 8. Read-only through the report; nudge and close comments are drafted for review and never posted without a per-item yes.

This **reverses the "considered and rejected" `pipeline` entry** in the README, which claimed `gh search prs --author @me` already covered it. Two things it didn't:

- **The recommended one-liner was broken.** README's "Checking your pipeline" snippet asked for `--json url,title,repository,reviewDecision,updatedAt`, and `gh search prs` has no `reviewDecision` field (nor `mergeable`, `statusCheckRollup`, or `mergedAt`) — it exits with `Unknown JSON field`. The state you actually want is only reachable through a second enrichment pass, so the "just use gh" position was resting on a command that never ran.
- **Bot noise makes the obvious signals useless.** On a real upstream PR the last commenter is nearly always a deploy-preview bot, CodeRabbit, a coverage bot or a security scanner, so both "last commenter isn't me" and `updatedAt` mis-report ball ownership. `mastra-ai/mastra#16639` reads as *updated today* off an automated platform approval while the last human word was weeks earlier, and shows four "failing" checks of which two are docs previews on an already-approved PR. The insight is in the noise filter, not the listing — which is what makes it skill-shaped rather than alias-shaped.

Design notes, all verified against the live `gh` CLI before shipping:

- **Bot detection needs a name list, not just `__typename`.** App-backed accounts (`dane-ai-mastra`, `superagent-security`) come back as `User`, not `Bot`, so `__typename` alone under-filters. The skill combines `__typename` / `[bot]` suffix / a profile-configurable ignore list / bot-template body shapes, and computes staleness from human events only.
- **`mergeStateStatus` is the authority, `statusCheckRollup` over-reports.** `DIRTY`/`BEHIND`/`BLOCKED`-with-required-failure is yours; `UNSTABLE` (non-required checks only) and `BLOCKED`-with-green-checks (waiting on review) are theirs. `mergeable` returns `UNKNOWN` on first ask because GitHub computes it lazily — the skill re-queries and reports "unknown" rather than "clean".
- **One batched GraphQL for enrichment**, aliased `repository { pullRequest(number:) }` per item sharing a fragment — the same shape `find-issues` Phase 3 uses — instead of N `gh pr view` round-trips.
- **Explicit `--author <login>`, never bare `@me`**, and no `gh auth switch` at all for the read-only phases: `@me` resolves against the active account and silently reports the wrong person's pipeline on a two-account machine.
- Issues already covered by your own linked PR collapse into one row instead of double-counting.
- Nudge policy: at most one per item per 14 days, and none at all if your own last comment was already a nudge — offer close/escalate/walk-away instead.

First live run (2026-08-01, 12 open PRs + 3 open issues) validated two rules the hard way. **All four** PRs that returned `mergeable: UNKNOWN` came back `CONFLICTING`/`DIRTY` on the mandated re-query — without it, a third of the pipeline reads as fine. And on `elie222/inbox-zero#2662` the only non-you comment in 79 days was `CLAassistant` (a `User`, not a `Bot`), so an unfiltered "someone replied, go answer them" would have been about a CLA badge. Conversely `mastra-platform`'s automated `APPROVED` on `mastra-ai/mastra#16639` *does* report `Bot` and is correctly filtered — that PR is approved by a robot and blocked on a required E2E job, not ready to merge.

### Added — `contribute-upstream` refuses to fix code nothing calls (Phase 3 step 4) and stops taking drive-by issues on faith (Phase 1 step 5b)

The tractability gate caught re-implementations and half-fixes but had no notion of *reachability*: a clean, well-tested patch to an orphaned module passed every gate the skill had. New third failure mode — **dead target** — requires proving the symbol is reachable from something a user runs (`git grep -n '<symbol>' -- . ':!*test*' ':!*spec*'`, or a repo code search pre-clone) before the fix is written. Definition plus its own tests only = switch to Phase 6 and ask whether the module is meant to be wired in.

New Phase 1 step 5b handles the upstream cause, for bugs that come from an issue the user didn't file: confirm the path the issue cites actually exists (a 404 means the issue was written against a different tree), and check whether the filer has exactly one issue each across a dozen unrelated repos with "Corpus reference:"-style boilerplate — a bulk automated scan whose premises are unverified by construction. Neither is a hard stop; both change how much of the issue you take on faith.

Documented case: `CopilotKit/CopilotKit#4842` sat 68 days before the maintainer replied that `fetchWithRetry` and its whole module are referenced only by their own test file — "until the retry utility is actually integrated, this is polishing dead code" — while calling the patch itself "clean and well-tested". The code was never the problem; the target was. The same PR also added a `.changeset/` file to a repo that had moved off changesets (`.changeset/` 404s on `main`), and the issue driving it cited `packages/runtime/src/util/retry-utils.ts`, a path that does not exist. Cost of the missing check: one `git grep`.

### Fixed — README's pipeline one-liner and stale skill counts

The broken `reviewDecision` snippet is replaced with a form that runs (`--json number,repository,title,updatedAt` + template; `--template` also requires `--json`, which the old snippet did satisfy). `CLAUDE.md`, `docs/profile.example.md`, and the install-verification note still said "three skills" after `log` shipped in v0.7.0; they now cover all five.

## [0.10.0] — 2026-06-09

### Fixed — `find-issues` duplicate-PR search now covers closed PRs and acts on the issue timeline it already fetches

The dup-PR search ranked a dead candidate as the #1 pick because it (a) searched `--state open` only and (b) leaned on code-identifier tokens that didn't appear in the competing PR's title. Three small changes to the Phase 3 duplicate-PR search and the Phase 4 freshness re-check:

- **Search all PR states, not just open.** Dropped the `--state open` filter from the token queries. A closed-not-merged PR for the exact fix is a *louder* signal than an open one and was previously invisible. This was the one true gap.
- **Act on the cross-reference the enrichment query already pulls.** The Phase 3 enrichment GraphQL already fetched `timelineItems(CROSS_REFERENCED_EVENT)`, which lists a linked PR in *any* state — but the drop-rule only looked at `closedByPullRequestsReferences`, which omits closed-unmerged PRs. Extended the enrichment timeline to also capture CONNECTED_EVENT + PR title/state, and broadened the drop-rule to act on any-state timeline cross-refs. The bare issue-number token search is marked mandatory on every candidate and on the Phase 4 re-check — code-identifier tokens may never substitute for it. (No new standalone query: the data was already in hand.)
- **Interpret non-open hits instead of ignoring them.** New triage: *merged but issue not auto-closed* → fix shipped, drop; *approach rejected by maintainer* → design-sensitive, demote/drop; *auto-closed by an anti-AI-PR bot* → repo has an anti-agent policy, drop the candidate and skip-list the repo; *abandoned by author* → re-openable, keep but disclose the prior attempt in the why-line. (`gh search prs --json state` returns `merged` as a distinct state, so the merged bucket costs no extra call.)
- **Repaired the prescribed commands the new checks depend on** (verified against the live `gh` CLI). The Phase 3 batch-enrichment GraphQL was syntactically invalid and declared an unused variable — and GitHub has no bulk issues-by-number field, so even repaired it would have fetched the wrong issues. Replaced with aliased `issue(number:)` fields sharing a fragment, plus a note to tolerate per-alias partial errors. `gh issue view --json timelineItems` is not a real field: the Phase 3 fallback now pairs `gh issue view` with the REST `issues/<n>/timeline` endpoint, and the Phase 4 re-check reuses the enrichment GraphQL instead.

Documented case: 2026-06-09 trending hunt ranked `vitest-dev/vitest#10491` #1. PR #10492 ("fix: correct multiple negated --project filters combination") had been opened and **auto-closed within ~2h** after vitest's bot labeled it `maybe automated` ("appears to have been fully generated by AI"). The `--state open` search and a `matchesProjectFilter`-token re-check both missed it; `closedByPullRequestsReferences` was empty but the enrichment timeline already carried the closed cross-ref. Bonus finding: vitest runs an anti-AI-PR auto-close bot and belongs on the skip list.

## [0.9.0] — 2026-05-24

### Added — `contribute-upstream` catches label-scoped invitation-only conventions (Phase 1 step 0c)

Some projects enforce internal-team-only policy **scoped to specific issue labels**, not to the whole repo or to prose in CONTRIBUTING. Existing Phase 1 step 2 detection (scan CONTRIBUTING / PR template for "invitation only" / "do not accept unsolicited" / etc.) reads docs that never mention the label-scoped rule — the policy is enforced by a maintainer closing external PRs with a brief "the team handles this label" comment.

New step **0c** runs after the existing freshness / adjacent-PR / already-fixed-on-main gates: read the issue's labels, then sample recent closed-not-merged PRs that linked an issue carrying each label. Two or more closures with matching closer-comment shapes ("the team handles bugs marked with `<label>`", "internal-team area / closed in favour of internal work", "cannot accept external PRs for `<label>` issues") = drop and switch to Phase 6.

One closure could be idiosyncratic. Two with matching wording is policy.

Documented case: 2026-05-22, `ChromeDevTools/chrome-devtools-mcp` PRs #2098 + #2099 closed same-day by the same maintainer with effectively identical wording — "difficulty here is not in generating a fix but running evals … team handles bugs marked with the `evals` label." Both PRs passed every other Phase 1 gate. Nothing in CONTRIBUTING or the PR template said so. The cheap query (`gh search prs --repo <r> --state closed "is:unmerged label:<label>"`) would have flagged the policy in one round-trip before clone.

### Added — `contribute-upstream` Phase 8 step 0 catches mid-review competing merges

PRs that sit in review for days are exposed to maintainers landing a competing fix on the same surface. When that happens, the PR is functionally obsolete and the right move is to close it gracefully, not to keep iterating on review comments. Existing Phase 8 jumped straight to fetching review state — if the file moved under us during the gap, classifying feedback against a dead PR was pure waste.

New step **0** in `references/phase-8-review.md` runs before any feedback classification: list commits to `main` on each file the PR touches, since the PR's `createdAt`. Three outcomes — competing fix shipped (close gracefully), file changed but addressing a different bug (rebase), or no movement (proceed to step 1).

Documented case: 2026-05-13 → 2026-05-22, `better-auth/better-auth#9605` (normalize SSO OIDC `?error=signup disabled` → `signup_disabled`) sat 9 days with zero human review. Maintainer landed `#9722` (URL-encode the value → `signup%20disabled`) on 2026-05-22 — different approach, same bug, same file. `#9605` became `mergeable_state: dirty` and functionally obsolete the moment `#9722` landed. The freshness gate at Phase 1 step 0 wouldn't have caught this because the competing PR didn't exist at open time; the Phase 8 review-response loop wouldn't catch it because there was no review to respond to. Step 0 catches the silent-then-obsolete shape.

## [0.8.0] — 2026-05-22

### Added — `find-issues --trending [daily|weekly]` for trending-repo discovery

A new flag promotes the previously-undocumented ad-hoc workflow (running trending hunts manually from a profile-skip-list pattern) into a first-class entry point with explicit drop rules.

The default mode's "no invented watchlist" hard rule was the right default — popular/curated repos are nearly always the highest-yield surface — but in practice the user runs trending hunts roughly weekly and the skill had no documented support for them. Past hunts were ad-hoc shell sessions that re-derived the same drop rules from the profile's skip list; this change captures those rules in the skill and adds two new ones surfaced by today's hunt.

**New Phase 2 sub-section "Trending-mode discovery"** runs *before* the per-repo issue-search fan-out:
- **Step T1** — fetch `github.com/trending/<lang>` (daily + weekly) via WebFetch since `gh` has no trending endpoint
- **Step T2** — de-dupe against the profile's watched and skip lists, cap survivor pool at ~7
- **Step T3** — aggressive repo-level gate (parallel batched) with three stricter drop rules
- **Step T4** — continue with the unchanged Phase 2 per-repo issue search

**Three trending-only drop rules** beyond the existing Hot/Active/Slow/Dormant/Invitation-by-fast-close tiers:

1. **Top-contributor ≥60% lockdown** (stricter than the default 70% on watched repos, because trending hunts have no prior trust signal).
2. **Team-only namespace lockdown** — verify via `gh api users/<login> --jq .email,.bio` on the top 2–3 if all logins share an org prefix.
3. **AI-bot-as-merge-author** — drop if any account ending in `-agent` / `-bot` / `[bot]` (excluding dependency bots like dependabot/renovate) appears in the top 5 merge-counted authors. Documented cases: `archestra-ai/archestra` (2026-05-22, `archestra-contributor-pr-bot` owned 73% of 30d merges) and `anomalyco/opencode` (2026-05-22, `chaodu-agent` appears in fast-close-rejected externals).

**Phase 5 addendum** surfaces *skip-list candidates* — repos that failed the trending gate — as a paste-ready block at the end of the hand-off. The skill never writes to the profile; the user pastes via `/oss-contribute:profile edit`. The hard-rules section now explicitly bans profile writes, including skip-list additions.

**Phase 5 issue-cluster hint** — when 3+ ranked candidates target the same surface (e.g. four `evaluate_script` tool-description bugs in `ChromeDevTools/chrome-devtools-mcp`, 2026-05-22), flag the bundling opportunity in one line. Don't decide for the user; surface it.

**Rate-limit hygiene** (new sub-section, applies to all modes but bites hardest in `--trending`) — the GitHub search API's secondary rate limit kicks in after ~10–30 rapid queries. Trending mode fans out across 7+ repos × ~3 queries each. Documented case from today's hunt: the 5th-7th repo gate calls returned `HTTP 403 secondary rate limit`. Pre-flight check (`gh api rate_limit --jq '.resources.search.remaining'`) and polling pattern (`until ... ; do sleep 15; done`) added.

Documented successful trending hunt (2026-05-22, TypeScript daily+weekly): 30 trending repos → 7 candidates after de-dupe → 1 survivor after gate (`ChromeDevTools/chrome-devtools-mcp`) → 5 ripe issues (`#1892`, `#1894`, `#1895`, `#1896`, `#1217`), with 4 of 5 in the same `evaluate_script` tool-description cluster. The gate rejected 6/7 candidates and the cluster-bundling hint surfaced naturally — both surfaces validated end-to-end.

## [0.7.6] — 2026-05-19

### Added — `contribute-upstream` Phase 1 step 0a now also catches *active* adjacent PRs that reshape a shared interface

The existing step 0a check only caught the "dead area" shape — an adjacent open PR with zero engagement sitting 30+ days. It missed the inverse: an *active* adjacent PR that reshapes the interface your fix calls into. Files don't textually overlap, so the token-based dup-PR search returns nothing; the risk is that the interface gets reshaped mid-review and your work has to either wait or chase a moving target.

Step 0a now describes both shapes — dead area (no engagement, stalled) and moving target (active, reshapes shared interface) — with separate cheap checks for each. Step renamed from "Adjacent-stalled-PR check" to "Adjacent-PR check" to reflect the broader scope.

Documented case: 2026-05-19, `topoteretes/cognee#2815` (small feature: plumb `node_name` through `ChunksRetriever`) passed every existing gate — freshness, token dup-PR search, "already fixed on main" — but adjacent open PR #2712 (`fix: implement include_payload and node_name filter in ChromaDBAdapter`) was reshaping the shared `vector_db_interface.py` mid-review: 4 reviews, 6 comments, CodeRabbit auto-paused for active development. Token dup search returned nothing because the PRs are about different layers (retriever vs adapter). Caught only by a manual Phase 1 scan; without that scan, the contributor would have shipped against an interface that may not exist post-#2712-merge.

## [0.7.5] — 2026-05-19

### Added — `find-issues` detects "invitation-by-fast-close" repos at Phase 2

A new failure mode the existing lockdown checks missed.

The existing repo-activity tiers (Hot/Active/Slow/Dormant) and the "≥3 distinct external authors merged in last 30d" filter are evaded by repos that look healthy on paper but enforce a policy where new-contributor PRs get auto-closed within minutes pending a maintainer `lgtmi`. The 30d-distinct-author count includes pre-approved contributors, so the repo passes the lockdown check; the user only discovers the policy after burning half a day on a PR that gets closed in 10 seconds.

The existing Phase 3 "Invitation-only upstream" drop searches for literal CONTRIBUTING phrases ("invitation only" / "do not accept unsolicited" / "closed without review"). That phrase-match misses repos that enforce the same policy without writing it in those exact words.

Two coupled changes:

**1. New "Invitation-by-fast-close" repo-activity tier** (Phase 2). When the distinct-author count looks healthy (≥10 in 30d) but a single top contributor still owns the bulk of merges, sample the last ~10 closed-not-merged PRs from non-top-contributors and inspect `closed_at - created_at`. If 3+ were closed within minutes-to-hours with no review iteration, drop the repo entirely. Internal-tracking labels like `closed-because-weekend`, `closed-because-refactor`, `possibly-X-clanker` are corroborating evidence.

**2. Strengthened Phase 3 "Invitation-only" drop** to also accept fast-close evidence even without the literal CONTRIBUTING phrase, with a cross-reference to the new Phase 2 tier.

Documented case: `earendil-works/pi` (2026-05-19) — 51k stars, 45 distinct merged authors in 30d (mitsuhiko top at ~17%), looked like the healthiest trending repo. In reality #4736/#4588 closed in 10s, #3517 closed in 2h, all from `NONE`-association authors. The 44 non-mitsuhiko PRs were pre-approved contributors. Scout subagent burned full enrichment on 30 candidates before realising none could land.

## [0.7.4] — 2026-05-17

### Changed — `contribute-upstream` stops re-prompting for the GitHub account AND now enforces a matching git commit identity

Two coupled changes for the same problem: getting the *commit attribution* right on OSS PRs without forcing the user to answer the same question every time.

**1. No more re-prompting for the GitHub account.** Phase 1 step 7 previously read the profile's `## Default GitHub account` only as a *proposed* default and then explicitly asked the user which account to fork from, on every single contribution. The justification was "do not silently use the profile default" — written for an early version where the wrong account could leak.

In practice this re-prompt fires on every run for users who have a stable mapping (e.g. personal `shiminshen` for all OSS, company `damonshen17` reserved for work). The profile entry IS the standing instruction. Asking again treats it as ephemeral.

New behaviour: read the profile default and **use it without asking** — even when `gh auth status` shows multiple accounts. State the chosen account in the Phase 1 step 8 summary so it remains visible. Re-prompt only if the profile has no default field at all, or if the user has corrected the account within the same session.

**2. New: set local git `user.name` / `user.email` on every clone (BLOCKING before commit).** Picking the right *GitHub account* for the PR is only half the answer. The other half is the *git commit identity* — `git config user.name` / `user.email` — which is typically inherited from the user's global config and is usually their work email (e.g. `damon@deeplearning.ai`). Without intervention, an OSS PR would appear under the personal GitHub account while every commit in `git log` is forever stamped with the work email. Exactly the wrong signal.

The skill now reads a new `## Git commit identity` profile section (name + email, suggested format: GitHub noreply like `<id>+<login>@users.noreply.github.com` to keep real email private) and runs `git -C <clone-dir> config user.name/email <profile values>` immediately after `git clone` / `gh repo clone`, before any commit. Phase 2 step 2 makes this BLOCKING — verify with a follow-up `git -C <clone-dir> config user.email` before proceeding.

If the profile lacks the identity section, ask the user explicitly — never silently inherit the global identity for an OSS clone.

Documented preference: the `oss-personal-account` memory captures the personal-vs-company split and the standing instruction. Example profile section users should add (see `docs/profile.example.md` if you want a template):

```
## Git commit identity
- name: shiminshen
- email: 16914659+shiminshen@users.noreply.github.com
```

## [0.7.3] — 2026-05-17

### Fixed — `find-issues` drops issues where the reporter has offered to open the PR themselves

New Phase 3 drop criterion: if the issue body contains a ready-to-apply diff PLUS a self-claim from the reporter ("Happy to open a PR", "I'll send a PR", evidence they already built+tested locally), drop the candidate. No assignee is set only because the reporter hasn't pushed the button yet.

The reactive path (`contribute-upstream`) would already catch this at Phase 1's freshness re-check via the issue body, but by then the user has invested attention in the candidate. Catching it in `find-issues` keeps the shortlist clean.

Documented case: `amruthpillai/reactive-resume#3077` (filed 2026-05-16 by `netooran`). Reporter wrote the complete root-cause analysis, posted a two-file diff in fenced ` ```diff ` blocks, and ended with "Happy to open a PR with the patch above. Verified the fix end-to-end" — but the scout's prior heuristics flagged this as a *positive* signal ("ready diff = best candidate, ~30 min fix"), ranking it #1. The user caught the trap at the `contribute-upstream` handoff. Three reasons to drop, not lift:

1. Community courtesy — beating the reporter to their own PR is rude.
2. Maintainer optics — visibly lifting another contributor's analysis reads as credit-stealing.
3. Portfolio quality — for users running `log`, a PR that visibly originated from someone else's diff is a worse signal than no PR.

The detection is cheap: scan the issue body for fenced ` ```diff ` blocks, "Proposed fix" sections authored by the reporter, or explicit offer phrases. Add to the Phase 3 drop list alongside the existing AI-bot, subsystem-stall, and invitation-only gates.

## [0.7.2] — 2026-05-16

### Fixed — `find-issues` triage gates strengthened by real-hunt failures

Three new drop conditions / token-search rules in `find-issues`, each from a documented failure in the 2026-05-16 trending hunt:

- **Module-augmentation tokens added to dup-PR search.** Token type 3 ("backticked code identifiers") now explicitly calls out import paths, augmented-module names from `declare module 'X' { ... }` blocks, and interface names being augmented (e.g. `Register`, `Routes`). Documented case: `TanStack/router#7399` ("server entry boilerplate gives type error") had an unmerged dup PR `#7357` ("fix(start): import Register from framework package so module augmentation works"). Title-paraphrase tokens (`server`, `boilerplate`) returned nothing; the dup was caught only when body tokens (`createServerEntry`, `Register`, `requestContext`) were tried. The shape — bug filed against documented module augmentation, fix described in terms of imports — is a class, not a one-off.

- **AI-bot triage poisoning is now a BLOCKING gate.** New Phase 3 drop criterion: comments from accounts ending in `-agent`, `-bot`, `[bot]`, or self-disclosed AI agents are treated as **hypotheses, not facts**. Verify the referenced PR exists and was merged via `gh pr view`, and check any version-number claim against the actual npm registry. Documented case: `vercel/ai#15302` had a `kagura-agent` comment correctly identifying merged PR #14102 as the fix but fabricating the shipped version (`@ai-sdk/google-vertex@4.1.12` — actual latest on npm: `4.0.130`). The PR was real; the "shipped in" claim was a hallucination that *sounded right*. Treating the comment as a fact would have dropped a candidate that was actually viable.

- **Subsystem stall** added as a drop criterion. When reaction-sorting surfaces an older bug (3+ months) with high engagement, check open PRs targeting the same file/subsystem; if ≥3 sit in `REVIEW_REQUIRED` with `created == updated` (opened then never iterated), drop. Whole-repo merge cadence is a misleading green light when a specific subsystem is dead. Documented case: `vercel/ai#6974` — maintainer @lgrammel personally reproduced 2025-08-29, root cause identified in-thread, but PRs #12875, #13209, #13851, #14689 all sat untouched since open-date despite the repo merging 50 PRs/week. Same shape as the profile-side `oss-subsystem-lockdown` lesson, hoisted into the skill itself so all users benefit.

The 2026-05-16 hunt also reaffirmed an existing rule: trending-repo discovery is high-effort, low-yield in this ecosystem. The user's profile already documents trending hunts as a false-positive pattern; this session added two more skip-list entries (`millionco/react-doctor`: single-maintainer lockdown — `aidenybai` 20/20; `payloadcms/payload`: team-only merge pattern — 8 distinct authors / 30d, all core team) which are personal to the watchlist and go in the user's profile, not here.

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
