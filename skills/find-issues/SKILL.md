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
/oss-contribute:find-issues --trending [daily|weekly]      # discover repos via github.com/trending (default: daily)
```

Stable preferences live in the shared profile (see "Profile location" below). Args are per-invocation overrides only. Cap at the flags above — if you need more knobs, edit the profile.

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

> **Trending mode** (`--trending`): expand the candidate pool with repos from `github.com/trending/<lang>` before per-repo issue search. See "Trending-mode discovery" below — it adds an extra repo-list-building step that runs *before* the per-repo subagents fan out. Everything after that is identical to the default path.

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
  --query 'repo:<owner>/<repo> state:open created:>=<date-7d> (label:"good first issue" OR label:"help wanted" OR label:"bug") sort:created-desc' \
  --limit 30
```

(Drop the `created:` filter in `--repo` focus mode, where older reaction-sorted bugs are fair game — the subsystem-stall gate in Phase 3 exists for those.)

Cap the total candidate pool at ~60 across all repos before triage, to keep Phase 3 cheap.

## Phase 3 — Triage each candidate

### Batch-fetch enrichment via GraphQL (use this, not N separate `gh issue view` calls)

`gh issue view --json` is one round-trip per candidate. For 30 candidates that's 30 round-trips. **Use one GraphQL call instead:**

GitHub has no bulk issues-by-number field — build the query with one aliased `issue(number:)` field per candidate, all sharing a fragment:

```
gh api graphql -F owner=<owner> -F repo=<name> -f query='
  query($owner:String!,$repo:String!){
    repository(owner:$owner,name:$repo){
      i<n1>: issue(number:<n1>){...F}
      i<n2>: issue(number:<n2>){...F}
    }
  }
  fragment F on Issue {
    number title body createdAt updatedAt
    assignees(first:5){nodes{login}}
    labels(first:20){nodes{name}}
    comments(last:3){nodes{author{login} body createdAt}}
    closedByPullRequestsReferences(first:5){nodes{number state}}
    timelineItems(last:10,itemTypes:[CROSS_REFERENCED_EVENT,CONNECTED_EVENT,ASSIGNED_EVENT]){
      nodes{__typename
        ... on CrossReferencedEvent{source{__typename ... on PullRequest{number state title createdAt}}}
        ... on ConnectedEvent{subject{__typename ... on PullRequest{number state title}}}}
    }
  }'
```

**Tolerate partial errors.** A number that resolves to a PR (or nothing) returns `null` for that alias plus an `errors` entry, and `gh` exits non-zero — but `data` for the other aliases still comes back on stdout. Parse the output regardless of exit code; treat the batch as failed only if `data` itself is absent. The point is **one round-trip, not N**.

**Fallback** if GraphQL is unavailable: dispatch the per-issue calls **in a single message**, one Bash tool call per candidate, batched in parallel. Not sequentially. `gh issue view --json` has **no `timelineItems` field** — pair it with the REST timeline endpoint to get the cross-referenced PRs:

```
gh issue view <n> --repo <owner>/<repo> --json \
  assignees,labels,comments,closedByPullRequestsReferences,body,createdAt,updatedAt
gh api repos/<owner>/<repo>/issues/<n>/timeline --jq \
  '[.[] | select(.event=="cross-referenced" and .source.issue.pull_request)
    | {number:.source.issue.number, state:.source.issue.state,
       merged:(.source.issue.pull_request.merged_at!=null), title:.source.issue.title}]'
```

Drop the candidate if **any** of these is true:

- It has an assignee.
- `closedByPullRequestsReferences` is non-empty, OR the enrichment `timelineItems` shows a cross-referenced/connected PR that is **open or merged**. (`closedByPullRequestsReferences` lists only PRs that *closed* the issue, so a dup that didn't auto-close it hides from that field but shows in the timeline.) A **closed-not-merged** cross-ref does not auto-drop — route it through the triage in "Duplicate-PR search" below, which decides.
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

**The issue's own timeline is the authoritative cross-reference** — it catches a PR whose title shares no tokens with the issue (anything that says "fixes #N" or is manually linked), in any state. You already have it from the Phase 3 enrichment `timelineItems` — don't re-query it here, just act on it. The token searches below are the *second* net, for PRs that fix the bug without referencing the issue at all:

1. **The issue number itself.** `"#93700"` and bare `"93700"` — many PR descriptions reference it. **This query is mandatory on every candidate, including the Phase-4 re-check — never substitute code-identifier tokens for it.**
2. **URL-encoded or other distinctive literals** in the issue body — `%5F`, error codes, magic strings.
3. **Backticked code identifiers** from the issue body — function names, file paths, type names (e.g. `` `LayoutRoutes` ``, `` `buildUpdateSet` ``). Also include: import paths, augmented-module names from `declare module 'X' { ... }` blocks, and interface names being augmented (e.g. `Register`, `Routes`). Module-augmentation bugs are paraphrased away by title-keyword search.
4. **Error message fragments** quoted in the body, if any.

**Search ALL states, not just open** — `gh search prs --repo <owner>/<repo> --limit 5 <token>` (no `--state` filter; add `--json number,title,state,createdAt` to read state). A `--state open`-only search is the bug that shipped a dead candidate: it silently skips a closed-not-merged PR for the exact fix, which is a louder signal than an open one (see below).

A title-keyword search alone is not enough. Documented failure cases:

- Issue titled "Layouts for paths that start with underscore (%5F)…" had an open PR titled "fix(typegen): normalize %5F to _…" — caught only by the `%5F` token search.
- `TanStack/router#7399` ("server entry boilerplate gives type error") had open PR `#7357` ("fix(start): import Register from framework package so module augmentation works"). Token search using title-paraphrases `server,boilerplate` returned nothing; the dup was caught only when the body tokens `createServerEntry`, `Register`, `requestContext` were tried.

**Interpreting a non-open hit.** `gh search prs --json state` returns `merged` as a distinct state, so merged vs. closed-not-merged is readable straight from the search results:

- **Merged** → the fix already shipped; the issue just didn't auto-close (no "fixes #N" keyword in the PR body). Drop.

If the PR is closed-and-unmerged, do NOT silently treat the issue as ripe — inspect *why* it closed with `gh pr view <pr> --repo <o>/<r> --json state,mergedAt,author,closedAt,comments,reviews,labels`:

- **Approach rejected by a maintainer** (review comment explaining a different design, or "we don't want this") → the issue is design-sensitive; demote to issue-comment-only or drop. A fresh PR with the same approach will be rejected too.
- **Auto-closed by an anti-AI-PR bot.** A bot/`github-actions` comment like *"labeled `maybe automated` because it appears to have been fully generated by AI"* or any "confirm you are a real human / disclose if automated" gate means the repo **auto-kills AI-generated PRs**. This is an anti-agent policy (same family as immich/transformers/playwright) — it is **near-fatal for this flow regardless of issue ripeness**. Drop the candidate AND surface the *repo* as a skip-list addition (anti-AI-PR auto-close). The issue may still be technically ripe for a human, but not for a Claude-driven PR.
- **Abandoned by author** (closed with no review, no bot, often by the author themselves) → genuinely re-openable; the issue can stay a candidate, but say so explicitly in the why-line so the user knows a prior attempt exists.

Documented failure case: 2026-06-09 trending hunt ranked `vitest-dev/vitest#10491` (negated `--project` filters) as the #1 pick. A `--state open` dup-search and a code-identifier-token re-check both missed PR **#10492** — opened and **auto-closed within ~2h** after vitest's bot labeled it `maybe automated` ("appears to have been fully generated by AI"). Caught only when the user asked "did someone already do this?" Two lessons: search all states (the dup was *closed*), and act on the enrichment timeline's cross-referenced PRs even when `closedByPullRequestsReferences` is empty. Bonus: vitest belongs on the skip list (anti-AI-PR auto-close bot).

If a token query or the timeline returns an open or merged PR, or a closed-not-merged PR that falls in the "rejected" or "anti-AI-bot" buckets above, drop the candidate.

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

### Trending-mode discovery (`--trending`)

When invoked with `--trending`, build the candidate-repo list from `github.com/trending` *before* the per-repo issue search. The rest of Phase 2/3 runs unchanged on that list. This is opt-in because trending hunts are **high-effort, low-yield** in most stacks — the user's profile typically records past trending hunts as false-positive patterns.

**Step T1 — Fetch trending HTML.** `gh` has no trending endpoint. Use WebFetch on both:

```
https://github.com/trending/<lang>?since=daily
https://github.com/trending/<lang>?since=weekly
```

Default `<lang>` from profile's `## Languages`. Default `since=daily` unless the user passed `weekly`. Ask the WebFetch model to return only `owner/repo — short description — stars` per line, no commentary.

**Step T2 — De-dupe and filter.** From the merged trending lists, drop:
- Anything in the profile's `## Watched repos` (already known).
- Anything in the profile's `## Skip list` (already known to be a lockdown).
- Single-owner-namespace repos that smell solo (`<one-name>/<project>` with no organisation backer) — defer to gate signals rather than auto-dropping, but prioritise obvious multi-org candidates first.
- Non-code repos (skill libraries, content registries, "awesome-X" lists).
- Language barriers if the description is unclear and the repo isn't your stack.

Cap the survivor pool at ~7 repos before gating. Trending is noisier than the watched list; over-fanning out burns rate limit fast.

**Step T3 — Aggressive repo-level gate** (one batched message, all repos in parallel). For each survivor, fetch `gh api repos/<o>/<r>` + `gh api -X GET search/issues -f q="repo:<o>/<r> is:pr is:merged merged:>=<date-30d>" -f per_page=50 --jq '.items[].user.login' | sort | uniq -c | sort -rn`. Apply the existing Hot/Active/Slow/Dormant/Invitation-by-fast-close tiers AND these stricter trending-only drops:

- **Top-contributor ≥60% lockdown.** Stricter than the default 70% on watched repos because trending hunts have no prior trust signal. If one human author (excluding bots) holds ≥60% of merged PRs in the last 30 days, drop.
- **Team-only namespace lockdown.** If all top-5 merge-counted humans share an `@<org>.com` email or all-`<org>`-prefixed logins (visible via `gh api users/<login> --jq .email,.bio` on the top 2–3), the repo is internal-engineering-only. Drop.
- **AI-bot-as-merge-author.** If any account ending in `-agent` / `-bot` / `[bot]` appears in the top 5 merge-counted authors (excluding dependabot/renovate which are dependency bots, not contribution bots) — e.g. `archestra-contributor-pr-bot`, `chaodu-agent` — the repo is gamified or LLM-flooded. Drop. Documented case: `archestra-ai/archestra` (2026-05-22), `anomalyco/opencode` (2026-05-22).

For each *dropped* repo, prepare a one-line skip-list candidate (`<owner>/<repo>` — short reason — date) to surface to the user at Phase 5. **Do not** write to the profile — this skill is read-only. The user copy/pastes into their profile via `/oss-contribute:profile edit`.

**Step T4 — Continue with Phase 2 issue search** on the survivors. Same per-repo subagent fan-out as the default path.

### Rate-limit hygiene (applies to all modes, but bites hardest in `--trending`)

The GitHub search API enforces a secondary rate limit that triggers after ~10–30 rapid queries. Trending mode fans out across 7+ repos × {repo metadata, merged-author search, fast-close sample}, so the budget evaporates quickly.

Before running the per-repo gate batch, check budget once: `gh api rate_limit --jq '.resources.search.remaining'`. If <15, poll (`until gh api rate_limit --jq '.resources.search.remaining' | awk '{exit !($1 > 15)}'; do sleep 15; done`) before fanning out. If the gate batch itself hits 403 (secondary rate limit), poll-and-retry only the failed queries — don't restart the whole batch.

## Phase 4 — Output

**Pre-output freshness re-check (BLOCKING).** Scout subagents may have run minutes ago and a PR can land in that window — your shortlist goes stale. Before emitting the ranked list, run in a single batched message: (1) per top candidate, the issue-number token search **across all states** (`gh search prs --repo <o>/<r> --limit 5 "<n>" --json number,title,state` — the mandatory token from Phase 3, NOT a code-identifier paraphrase); (2) one Phase-3 enrichment GraphQL call covering all top candidates — it re-fetches assignees, state, `closedByPullRequestsReferences`, AND the timeline cross-refs that catch a closed-unmerged dup. (`gh issue view --json` has no `timelineItems` field — use the GraphQL, or the REST timeline fallback from Phase 3.) Drop any candidate where a new open PR appeared, or where a closed-not-merged PR falls in the "rejected"/"anti-AI-bot" buckets. Documented failures: 2026-05-13 — `mastra-ai/mastra#16514` passed the scout's dup-PR check; PR #16545 opened ~10 minutes later. 2026-06-09 — `vitest-dev/vitest#10491` shipped as the #1 pick because the re-check used a code-identifier token (`matchesProjectFilter`) under `--state open` and missed already-closed AI-PR #10492; the issue-number/all-states/timeline check would have caught it. Both surfaced only when the user asked "did someone already do this?"

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

**Trending-mode addendum.** After the ranked list, surface any skip-list candidates collected in Step T3 as a separate block:

```
Suggested skip-list additions (paste into profile via /oss-contribute:profile edit):
- <owner>/<repo>  # <reason — date>
```

Be terse and accurate — the user audits and pastes manually. Do **not** modify the profile from this skill.

**Issue-cluster hint.** When the top-5 contains 3+ candidates targeting the same file or tool surface (e.g. four `evaluate_script` tool-description bugs in `chrome-devtools-mcp`, 2026-05-22), call it out in one line: "Issues #X/#Y/#Z share <surface> — one PR could close all three." Don't decide for the user; just flag the opportunity. Bundled PRs read better to maintainers when the bugs are genuinely sibling-shaped.

## Hard rules

- **Read-only.** Never open issues, PRs, or comments from this skill. Writes only happen via `contribute-upstream` after the user picks a candidate. **Never write to the profile** — including skip-list additions discovered in trending mode. Surface, the user pastes.
- **No invented watchlist** (default mode). Use only what's in the profile (or supplied via `--repo` or `--trending`).
- **Trending mode surfaces, never replaces.** `--trending` augments the candidate pool for one invocation. It does not edit the watched list.
- **No auto-handoff.** Always pause for the user to pick.
- **Surface, don't pad.** If nothing is ripe today, say so. A short honest list beats a long padded one. Trending hunts especially: 0 ripe survivors is the most common outcome — say so plainly.
