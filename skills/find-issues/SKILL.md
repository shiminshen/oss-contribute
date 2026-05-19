---
name: find-issues
description: Find ripe open-source issues you could actually contribute to — runs across the user's watched-repo list, filters by language/stack/budget, scores each candidate by "is this ripe" signals (no assignee, no linked PR, small scope, recent maintainer activity), and outputs a ranked shortlist with a one-line "why this one". Hands off to /oss-contribute:contribute-upstream once the user picks one.
trigger: /oss-contribute:find-issues
---

# find-issues

The proactive sibling of `contribute-upstream`. Use when the user wants to **shop for** an OSS contribution rather than fix a specific bug they hit. The reactive path is `contribute-upstream` — when in doubt, suggest that one instead, because the user-brought-bug signal beats any ranking heuristic.

## Usage

```
/oss-contribute:find-issues                                # use profile defaults
/oss-contribute:find-issues --lang typescript              # filter by language
/oss-contribute:find-issues --repo better-auth/better-auth # focus on one repo
/oss-contribute:find-issues --budget 30m | 1h | weekend    # only suggest issues that fit
/oss-contribute:find-issues --refresh                      # ignore any cached results
```

Stable preferences live in the shared profile (see "Profile location" below). Args are per-invocation overrides only. Cap at the four flags above — if you need more knobs, edit the profile.

## Profile location

Read the profile in this order:

1. `$CLAUDE_PLUGIN_DATA/profile.md` — when running as an installed plugin (Claude Code sets this env var)
2. `~/.claude/plugins/data/oss-contribute/profile.md` — fallback for direct-installed plugins
3. `~/.claude/skills/oss-contribute/profile.md` — fallback for local-development mode

If none exist, dispatch to the `profile` skill to set one up before continuing.

## Phase 1 — Load (or create) the profile

If the profile is missing, dispatch to the `profile` skill's interactive setup, then resume.

The profile covers:

- **Watched repos** — explicit `owner/repo` list. No "popular repos" autodiscovery; the user names them.
- **Languages / stacks** — e.g. `typescript`, `go`, `python`, plus framework expertise.
- **GitHub account** — which logged-in `gh` account to fork/PR from.
- **Default budget** — `30m`, `1h`, `half-day`, `weekend`.
- **What "ripe" means** — heuristic seed for ranking.

## Phase 2 — Discover candidates

**Dispatch one `general-purpose` subagent per repo in a single message.** Not one-at-a-time — one message containing N parallel `Agent` tool calls, where N = number of watched repos. The instructions read sequentially but the calls must be batched; otherwise the model often serialises them.

Each subagent: fetches recent open issues (last 7d, sorted by created), filters to bugs with 0–1 comments, runs the token-based dup-PR search from Phase 3 against each, and returns a compact list (≤5 candidates × ≤3 lines each) with verdict. Aggregate in the main agent. This keeps raw `gh search issues` JSON out of the main context window.

Per-repo subagent prompt skeleton:
- Repo: `<owner>/<repo>`
- Profile stack: `<langs/frameworks from profile>` (only for relevance scoring; do not filter by GFI/HW labels — see profile rule)
- Return: top 5 ripe candidates (no dup PRs by token search, no assignees, ≤1 comment, opened in last 7d), one line each with `repo#N — title — why-ripe`. If 0 ripe, say "0 ripe in this repo".

### Single-query label search (use this, not N separate `--label` calls)

`gh search issues --label` is AND, not OR. **Do not** run one call per label and merge — that's 3× the API round-trips. Use `--query` with an OR expression so the labels collapse into a single search:

```
gh search issues --json number,title,labels,assignees,createdAt,updatedAt,url \
  --query 'repo:<owner>/<repo> state:open (label:"good first issue" OR label:"help wanted" OR label:"bug") sort:updated-desc' \
  --limit 30
```

Cap the total candidate pool at ~60 across all repos before triage, to keep Phase 3 cheap.

## Phase 3 — Triage each candidate

### Batch-fetch enrichment via GraphQL (use this, not N separate `gh issue view` calls)

`gh issue view --json` is one round-trip per candidate. For 30 candidates that's 30 round-trips. **Use one GraphQL call instead:**

```
gh api graphql -F owner=<owner> -F repo=<name> \
  -F numbers='[<n1>,<n2>,...]' -f query='
  query($owner:String!,$repo:String!,$numbers:[Int!]!){
    repository(owner:$owner,name:$repo){
      issues:nodes:issues(first:50){nodes{
        number title body createdAt updatedAt
        assignees(first:5){nodes{login}}
        labels(first:20){nodes{name}}
        comments(last:3){nodes{author{login} body createdAt}}
        closedByPullRequestsReferences(first:5){nodes{number state}}
        timelineItems(last:10,itemTypes:[CROSS_REFERENCED_EVENT,ASSIGNED_EVENT]){
          nodes{__typename ... on CrossReferencedEvent{source{__typename ... on PullRequest{number state}}}}
        }
      }}
    }
  }'
```

(Alias each candidate via `node(id:...)` if GitHub's bulk-by-number is unavailable in your CLI version — the point is **one round-trip, not N**.)

**Fallback** if GraphQL is unavailable: dispatch the per-issue `gh issue view --json` calls **in a single message**, one Bash tool call per candidate, batched in parallel. Not sequentially.

```
gh issue view <n> --repo <owner>/<repo> --json \
  assignees,labels,comments,closedByPullRequestsReferences,timelineItems,body,createdAt,updatedAt
```

Drop the candidate if **any** of these is true:

- It has an assignee.
- `closedByPullRequestsReferences` is non-empty (a PR is already linked).
- An open PR for the same fix exists (see "Duplicate-PR search" below).
- The most recent maintainer comment says "we're working on this" / "PR incoming".
- It's labelled `needs: info`, `wontfix`, `discussion`, `rfc`, or similar non-actionable.
- It's older than 6 months with no activity.
- **An AI-bot comment claims it's already fixed.** Treat comments from accounts ending in `-agent`, `-bot`, `[bot]`, or self-disclosed AI agents as **hypotheses, not facts**. Verify: (a) the referenced PR exists and is merged via `gh pr view`, (b) any version number claim matches the actual npm registry (`npm view <pkg> version`). Bots fabricate version numbers that *sound right*. Documented case: `vercel/ai#15302` had a `kagura-agent` comment correctly identifying merged PR #14102 as the fix but fabricating the version (`@ai-sdk/google-vertex@4.1.12` — actual latest on npm: `4.0.130`). Drop the candidate only after confirming the fix is shipped; if the bot is wrong about shipping, the bug may still be live.
- **Reporter has offered to open the PR themselves** (the "reporter-offered-patch" trap). When the issue body contains a ready-to-apply diff plus a self-claim like "Happy to open a PR", "I'll send a PR", "I'll open a PR shortly", or evidence the reporter already built+tested the fix locally — drop. No assignee is set only because the reporter hasn't pushed the button yet. Lifting their analysis + patch and beating them to the PR is rude in the community, reads as credit-stealing to maintainers, and produces a portfolio entry that's visibly someone else's work. Scan the issue body for: a full diff in fenced ` ```diff ` blocks, a "Proposed fix" section authored by the reporter, or explicit offer phrases. Documented case: `amruthpillai/reactive-resume#3077` (2026-05-16, reporter `netooran` filed a complete two-file diff + said "Happy to open a PR with the patch above. Verified the fix end-to-end") — surfaced only at Phase 1 of `contribute-upstream` after the scout had already shortlisted it as "ready diff = best candidate". Catch this in `find-issues`, not later.
- **Subsystem stall.** When reaction-sorting surfaces an older bug (3+ months old, high engagement), check for open PRs targeting the same file/subsystem. If ≥3 of them are in `REVIEW_REQUIRED` state with `created == updated` (opened then never iterated), the subsystem is review-stalled — the maintainers are merging *other* areas but leaving this one to rot. Whole-repo merge cadence (50 PRs/7d) can be high while the specific subsystem is dead. Drop. Documented case: `vercel/ai#6974` (controller-close race on stream resume, maintainer @lgrammel reproduced 2025-08-29) — 4 separate PRs (#12875, #13209, #13851, #14689) all sat untouched since open-date.
- **Scope/intent ambiguity** (feature gone, not bug). For issues filed against a `vN.0.0-beta/rc.M` of a package mid-rewrite, briefly check whether the missing thing exists in the new code paths. If wholly absent, demote to issue-comment-only. Full criteria + documented case (`drizzle-orm#5755`) in `contribute-upstream` Phase 3 step 4.
- **Invitation-only upstream.** Drop if the upstream's contributing doc or PR template contains "invitation only" / "do not accept unsolicited" / "closed without review". Full check + documented case (`openai/codex`) in `contribute-upstream` Phase 1 step 2. **Also drop on invitation-by-fast-close evidence even without the literal phrase** — see Phase 2 "Invitation-by-fast-close" tier; documented case `earendil-works/pi` (2026-05-19).

### Duplicate-PR search

Issue-title keywords miss PRs whose title describes the *implementation* rather than the *symptom*. Search by tokens extracted from the issue body, not by paraphrases of the title.

**Dispatch all token queries in parallel.** Send ONE message containing one Bash tool call per token type — not 4–5 sequential calls. Inspect all results together and short-circuit if any returns a PR hit.

The four token types per candidate:

1. **The issue number itself.** `"#93700"` and bare `"93700"` — many PR descriptions reference it.
2. **URL-encoded or other distinctive literals** in the issue body — `%5F`, error codes, magic strings.
3. **Backticked code identifiers** from the issue body — function names, file paths, type names (e.g. `` `LayoutRoutes` ``, `` `buildUpdateSet` ``). Also include: import paths, augmented-module names from `declare module 'X' { ... }` blocks, and interface names being augmented (e.g. `Register`, `Routes`). Module-augmentation bugs are paraphrased away by title-keyword search.
4. **Error message fragments** quoted in the body, if any.

Each query: `gh search prs --repo <owner>/<repo> --state open --limit 5 <token>`.

A title-keyword search alone is not enough. Documented failure cases:

- Issue titled "Layouts for paths that start with underscore (%5F)…" had an open PR titled "fix(typegen): normalize %5F to _…" — caught only by the `%5F` token search.
- `TanStack/router#7399` ("server entry boilerplate gives type error") had open PR `#7357` ("fix(start): import Register from framework package so module augmentation works"). Token search using title-paraphrases `server,boilerplate` returned nothing; the dup was caught only when the body tokens `createServerEntry`, `Register`, `requestContext` were tried.

If any PR query hits, drop the candidate.

For each survivor, score on:

- **Repro quality** — does the body have steps + expected vs actual?
- **Scope estimate** — `one-liner` / `small` / `medium` / `large`. Drop `large` from a "weekend"-budget shortlist.
- **Stack match** — overlap with the user's languages/frameworks.
- **Repo activity (merge-likelihood)** — see below. Boost candidates from repos that actively merge PRs.
- **Maintainer signal** — has a maintainer triaged/labelled it?

### Repo activity scoring (merge-likelihood)

Issues in repos that don't merge anything are dead ends. Issues in repos that merge weekly are bets worth making. Fetch repo-level signals **once per watched repo**, in parallel with Phase 2's issue search — not per-candidate.

For each watched repo, one extra parallel call:

```
gh api repos/<owner>/<repo> --jq '{pushedAt, stargazersCount, openIssues:open_issues_count}'
gh search prs --repo <owner>/<repo> --state merged --merged ">=$(date -v-30d +%Y-%m-%d)" --limit 1 --json url
```

Combine into a per-repo activity tier:

- **Hot** — ≥1 PR merged in last 7 days AND `pushedAt` within last 14 days. **Strong boost.**
- **Active** — ≥1 PR merged in last 30 days. Normal weight.
- **Slow** — 0 PRs merged in last 30 days but the repo isn't archived. Demote one tier in the final ranking.
- **Dormant** — no merges in 30 days AND no commits to default branch in 60 days. Drop entirely; surface to user with "0 ripe (repo dormant)" so they can prune the watchlist.
- **Invitation-by-fast-close** — when distinct-author count looks healthy (≥10 in 30d) but a top contributor still owns the bulk of merges, sample the last ~10 *closed-not-merged* PRs from non-top-contributors. If 3+ were closed within minutes-to-hours of open with no review iteration, the repo is invitation-only in policy even if CONTRIBUTING doesn't use the literal phrase. Fetch with `gh api -X GET search/issues -f q="repo:<o>/<r> is:pr is:closed -is:merged -author:<top>" -f per_page=20` and check `closed_at - created_at`. Drop entirely; surface as "0 ripe (invitation-by-fast-close)". Documented case: `earendil-works/pi` (2026-05-19) — 45 distinct merged authors, but #4736/#4588 closed in 10s, #3517 closed in 2h, all from `NONE`-association authors. Internal-tracking labels like `closed-because-weekend`, `closed-because-refactor`, `possibly-X-clanker` are corroborating evidence.

Stars are a weak popularity signal — use only as a tiebreaker between two same-tier candidates, never as a primary score. A 1k-star repo with 5 merged PRs/week beats a 50k-star repo that merges nothing.

Cache the per-repo signals for the duration of one `find-issues` invocation (don't re-fetch across the Phase 2 ↔ Phase 3 boundary).

## Phase 4 — Output

**Pre-output freshness re-check (BLOCKING).** Scout subagents may have run minutes ago and a PR can land in that window — your shortlist goes stale. Before emitting the ranked list, re-run the Phase 3 duplicate-PR search against the top-5 candidates in a single batched message, plus `gh issue view <n> --json closedByPullRequestsReferences,assignees,state`. Drop any candidate where a new PR has appeared. Documented failure: 2026-05-13 hunt — `mastra-ai/mastra#16514` passed the scout's dup-PR check; PR #16545 opened ~10 minutes later. Caught only when the user explicitly asked "did you check no existing PR?"

Show the top 5 as a compact ranked list:

```
1. better-auth/better-auth#9412 — OIDC error encoded as "signup disabled" not "signup_disabled"
   Why: 5-line fix likely, clear repro, no assignee, your TS+auth stack. ~15 min.

2. ...
```

Stop after 5. If fewer than 3 survive Phase 3, say so and suggest broadening the budget or adding repos to the profile — don't pad with weak candidates.

## Phase 5 — Hand off

Ask the user to pick one. On pick:

- Open the issue in the browser: `gh issue view <n> --web`.
- Suggest `/oss-contribute:contribute-upstream <package>` to start the reactive flow with that issue as context. Do **not** auto-invoke.

## Hard rules

- **Read-only.** Never open issues, PRs, or comments from this skill. Writes only happen via `contribute-upstream` after the user picks a candidate.
- **No invented watchlist.** Use only what's in the profile (or supplied via `--repo`).
- **No auto-handoff.** Always pause for the user to pick.
- **Surface, don't pad.** If nothing is ripe today, say so. A short honest list beats a long padded one.
