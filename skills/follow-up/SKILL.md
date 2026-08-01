---
name: follow-up
description: Triage everything you have open upstream — authored PRs and authored issues — into "the ball is with you" vs "the ball is with them". Detects changes-requested, failing required checks, conflicts, maintainer questions, stale-bot auto-close warnings, and superseded work. Read-only by default; hands off to /oss-contribute:contribute-upstream Phase 8 for the items that need a push, and never posts a comment or closes anything without explicit confirmation.
trigger: /oss-contribute:follow-up
---

# follow-up

Answers one question: **of everything I've opened upstream, what is waiting on me?**

`gh search prs --author @me --state open` lists your open PRs. It does not tell you which ones are stuck on *you*, and the naive reading of "last commenter isn't me" is wrong on almost every real PR — deploy-preview bots, CodeRabbit, coverage bots and security scanners are the last commenter on most upstream PRs. This skill separates signal from bot noise, then sorts by who owes the next move.

**Read-only by default.** Phases 1–5 only query. Phase 6 (comment, close, re-push) is opt-in, per item, and gated on you seeing the exact text first.

## Usage

```
/oss-contribute:follow-up                      # everything open, default account
/oss-contribute:follow-up --repo owner/repo    # scope to one repo
/oss-contribute:follow-up --prs-only           # skip authored issues
/oss-contribute:follow-up --issues-only        # skip authored PRs
/oss-contribute:follow-up --stale 21d          # "waiting on them" threshold (default 14d)
/oss-contribute:follow-up --closed 30d         # recently-resolved window (default 14d)
/oss-contribute:follow-up --account <gh-login> # override profile account
```

## Profile location

Same three-tier resolution as every skill in this plugin — see `log` → "Profile location". Optional profile section, read when present:

```markdown
## Follow-up
- stale window: 14d
- ignore comment authors: vercel, coderabbitai, superagent-security
```

Missing profile is not fatal here: fall back to the active `gh` account and the defaults above, and say which account you used.

## Phase 1 — Resolve the account

Account resolution is canonical in `log` → Phase 1 (profile → `--account` → ask when ambiguous). One difference: **do not `gh auth switch` for the read-only phases.** `--author <login>` works regardless of which account is active, so Phases 2–5 need no switch and cannot leave the user's shell on the wrong account. Switch only in Phase 6, immediately before a write, and switch back after.

`--author "@me"` resolves against the *active* account — use the explicit login instead, or a two-account machine silently reports the wrong person's pipeline.

## Phase 2 — Inventory (two calls, no per-item queries yet)

```bash
gh search prs --author <login> --state open --sort updated --order desc --limit 100 \
  --json url,number,repository,title,createdAt,updatedAt,isDraft,commentsCount,labels

gh search issues --author <login> --state open --sort updated --order desc --limit 100 \
  --json url,number,repository,title,createdAt,updatedAt,commentsCount,labels
```

Then the resolved-since sweep (`--closed` window, default 14d):

```bash
gh search prs --author <login> --state closed --updated ">=$WINDOW_START" --limit 50 \
  --json url,number,repository,title,state,updatedAt
```

Verified `gh` facts — do not "improve" these into commands that don't exist:

- **`gh search prs --json` has no `reviewDecision`, `mergeable`, `statusCheckRollup`, or `mergedAt`.** Its field list is `assignees, author, authorAssociation, body, closedAt, commentsCount, createdAt, id, isDraft, isLocked, isPullRequest, labels, number, repository, state, title, updatedAt, url`. Everything else comes from Phase 3.
- **`state` on a closed PR *does* distinguish `merged` from `closed`** — the closed sweep needs no second query to split them. (`is:unmerged` as a bare query term also works if you want only the rejected ones.)
- `gh search issues` excludes PRs already; no `isPullRequest` filtering needed.
- `--state merged` is not valid on `gh search prs` (open|closed only).

**Drop your own repos.** Search returns items in repos you own (`<login>/*`) alongside upstream ones; they aren't contributions and they bury the signal. Filter them out unless `--repo` names one explicitly, and say how many you dropped.

If a `--limit` ceiling is hit, say so — do not silently truncate the pipeline you are reporting on.

## Phase 3 — Enrich (one batched GraphQL, not N view calls)

GitHub has no bulk-by-number field, so build one query with an aliased `repository(...) { pullRequest(number:) }` per item, all sharing a fragment — the same shape `find-issues` Phase 3 uses:

```graphql
fragment prBits on PullRequest {
  number url title isDraft state updatedAt reviewDecision mergeable mergeStateStatus
  headRefName baseRefName
  labels(first: 20) { nodes { name } }
  commits(last: 1) { nodes { commit { committedDate
    statusCheckRollup { state contexts(last: 30) { nodes {
      __typename
      ... on CheckRun { name conclusion detailsUrl }
      ... on StatusContext { context state targetUrl } } } } } } }
  comments(last: 10) { nodes { author { login __typename } authorAssociation createdAt body } }
  reviews(last: 10) { nodes { author { login __typename } state submittedAt body } }
  timelineItems(last: 20, itemTypes: [CROSS_REFERENCED_EVENT, CONNECTED_EVENT]) {
    nodes { __typename ... on CrossReferencedEvent { createdAt
      source { ... on PullRequest { number title state url author { login } } } } } }
}
query {
  p0: repository(owner: "<owner>", name: "<repo>") { pullRequest(number: <n>) { ...prBits } }
  p1: ...
}
```

Issues use the same shape with an `issueBits` fragment on `Issue` — same `labels` / `comments` / `timelineItems` selections plus `state stateReason`, minus everything merge-related.

Notes from running this live:

- `mergeStateStatus` works through `gh api graphql` with no preview header.
- Tolerate per-alias errors: one 404 (repo renamed, PR deleted) returns `data` for the rest alongside an `errors` array. Report the failed alias, keep the others.
- **`mergeable` is computed lazily and returns `UNKNOWN` on first ask.** Re-query the UNKNOWN subset once. If still UNKNOWN, report "conflict status unknown" — never report "no conflicts" from an UNKNOWN. This is not a corner case: on the first live run of this skill, **all four** `UNKNOWN` PRs came back `CONFLICTING`/`DIRTY` on the second ask. Skipping the re-query would have silently under-reported a third of the pipeline as fine.
- Fallback if GraphQL is unavailable: `gh pr view <url> --json reviewDecision,mergeable,mergeStateStatus,statusCheckRollup,reviews,comments,commits,labels` per item, dispatched **in one message** as parallel Bash calls, not sequentially.

## Phase 4 — Classify

### Step 1 — strip bot noise (do this before any "who spoke last" reasoning)

Treat a comment or review as noise when **any** holds:

- `author.__typename == "Bot"`, or the login ends in `[bot]`
- login is in the profile's `ignore comment authors` list, or matches known upstream noise: `vercel`, `netlify`, `coderabbitai`, `codecov`, `github-actions`, `changeset-bot`, `socket-security`, `sonarcloud`, `greptile`, `CLAassistant`
- the body is a bot template regardless of author: a deploy-preview URL table, a coverage delta, `Summary by CodeRabbit`, a security-scan verdict

**App-backed bot accounts do not report `__typename: Bot`.** Observed live on `mastra-ai/mastra#16639`: `dane-ai-mastra` and `superagent-security` both come back as `User`, as does `CLAassistant` on `elie222/inbox-zero#2662` — where it was the *only* non-you comment on the PR, so a naive filter reports "a human replied, answer them" about a CLA badge. Name list plus body shape is required, not belt-and-braces.

The reverse also happens: a platform auto-approval (`mastra-platform` on `mastra-ai/mastra#16639`) *does* report `Bot`, and filtering it is correct — an automated `APPROVED` is not a maintainer signing off.

What remains is *human* activity. Compute, per item: `lastHuman` (latest non-noise comment/review by anyone but you) and `lastYou` (your latest comment, review reply, or commit).

### Step 2 — read the merge state before the check rollup

`mergeStateStatus` is the authority on whether a PR is actually blocked; `statusCheckRollup` over-reports.

| `mergeStateStatus` | Meaning | Ball |
|---|---|---|
| `DIRTY` | Merge conflicts | **You** — rebase |
| `BEHIND` | Base moved, update required by repo policy | **You** — update branch |
| `BLOCKED` + a *required* check failing | Required CI red | **You** — fix |
| `BLOCKED` + checks green/pending | Waiting on required review | Them |
| `UNSTABLE` | Only non-required checks failing | Them (mention in one clause, don't escalate) |
| `CLEAN` | Mergeable, nothing owed | Them |
| `DRAFT` | Your draft | **You** — finish or close |

When the rollup is `FAILURE`, name the failing contexts before calling it CI-red. Preview-deploy contexts (`Vercel – <project>`, `Netlify`, docs builds) are almost never required — `mastra-ai/mastra#16639` sits at `APPROVED` + `BLOCKED` + four "failing" contexts, two of which are docs previews. Also compare the check timestamp against `commits.nodes[0].commit.committedDate` and the latest approval: checks that predate the current head or a later approval usually just need a re-run, which is a different ask than "your fix is broken".

### Step 3 — bucket every item

Order of evaluation matters — first match wins.

1. **At risk — deadline running.** A stale-bot warning comment or a `stale` / `no-response` / `pending-close` label with an auto-close deadline. Surface at the top regardless of everything else; these die from silence.
2. **Superseded.** A timeline cross-ref to a *merged* PR by someone else covering your surface, or (for issues) `closedByPullRequestsReferences` / a merged linked PR. Your work is probably dead — the move is close-with-thanks or rebase-and-narrow, not another review round. Same shape as `contribute-upstream` Phase 8 step 0; reuse its per-file commit check before you conclude it.
3. **Changes requested.** `reviewDecision == CHANGES_REQUESTED`, or a human review/comment with actionable feedback that postdates `lastYou`. → `contribute-upstream` Phase 8.
4. **Blocked by merge state.** `DIRTY` / `BEHIND` / `BLOCKED`-with-required-failure, per the table above.
5. **Question awaiting your answer** (mostly issues). `lastHuman > lastYou` and the comment asks something, or the item carries `needs-info` / `awaiting-response` / `needs-repro`. A maintainer asking for a version number and getting silence is the most common way a good issue dies.
6. **Your draft, gone cold.** `isDraft` and no activity from you in the stale window.
7. **Waiting on them, stale.** No human non-you activity for longer than the stale window (default 14d). Nudge candidate.
8. **Waiting on them, healthy.** Inside the window. One line, collapsed.

**Pair items, don't double-count.** An issue you filed that already has your own PR linked (`vercel/eve#705` → your `#706`) is one row, reported under the PR. Show the issue separately only when it needs something the PR doesn't.

Compute staleness from **human** activity, not `updatedAt`. A bot comment or a platform auto-approval refreshes `updatedAt` and makes a three-month-dead PR look active — `mastra-ai/mastra#16639` reads as updated today off an automated approval while the last human word was weeks earlier.

## Phase 5 — Output

Compact, bucketed, action-first. One line per item plus a *why*, and the exact next command:

```
Ball is with you (4)

  🔴 mastra-ai/mastra#16639   required E2E failing since Jun 26, approved Jul 30
                              → checks predate approval; re-run or rebase
  🔴 vercel/eve#856           changes requested by <maintainer>, 6d ago
                              → /oss-contribute:contribute-upstream https://github.com/vercel/eve/pull/856
  🟡 pnpm/pnpm#11868          stale-bot warning, auto-closes in 3d
  ⚪ better-auth#9923         your draft, untouched 21d — finish or close

Ball is with them (5)
  drizzle-team/drizzle-orm#5766   no human activity 78d — nudge candidate
  ... (collapsed one-liners)

Resolved since <date> (2)
  ✅ merged  better-auth/better-auth#10040
  ✖  closed unmerged  mastra-ai/mastra#16681
```

Rules for the report:

- **Say what you couldn't determine.** UNKNOWN mergeability, a failed GraphQL alias, a hit `--limit` ceiling — state it. A pipeline report that quietly omits an item is worse than no report.
- **No invented urgency.** "Stale" is a threshold, not a judgement. Don't tell the user a maintainer is ignoring them.
- If everything is healthy, say so in one line. Don't manufacture action items.

## Phase 6 — Act (opt-in, per item, confirmation-gated)

Only after the user picks an item. Never batch-apply across items.

| Bucket | Handoff |
|---|---|
| Changes requested | `contribute-upstream` Phase 8 (`references/phase-8-review.md`) — it owns classify → apply → reply → re-request |
| Conflicts / behind | `contribute-upstream` Phase 8, which runs the competing-merge re-check before any rebase |
| Superseded | Draft a close comment thanking the maintainer and pointing at the merged PR. Show it. `gh pr close` only on explicit yes |
| Stale, nudge | Draft a ≤3-line comment: what's blocked, what you need, an offer to close if it's not wanted. Show it. `gh pr comment` only on explicit yes |
| Question awaiting you | Draft the answer from what's in the repo. Show it. Post only on explicit yes |

**Nudge policy.** At most one nudge per item per 14 days, and if your own last comment *was* a nudge, don't offer another — offer the alternatives instead (close it, raise it in the issue, walk away). A second unanswered ping reads as pressure and costs you the next PR's goodwill.

Switch accounts (`gh auth switch -u <login>`) immediately before a write and switch back after, including on failure.

## Hard rules

- **Read-only through Phase 5.** No comment, close, reopen, label, push, or force-push happens without the user seeing the exact text/diff and saying yes, per item.
- **Never auto-nudge**, never post on a schedule, never "keep the PR warm" with a filler comment.
- **No AI-attribution** in any drafted comment — the plugin's global rule applies to nudges and close comments too.
- **Never report a conflict state from `mergeable: UNKNOWN`.** Re-query, then say "unknown".
- **Bot activity is not activity.** Staleness and ball-ownership are computed from human events only.
- **Explicit `--author <login>`,** never bare `@me` on a machine with more than one account.
- **Don't re-litigate merged work.** The resolved-since section is a receipt list; portfolio narrative is `log`'s job.

## When this skill is the wrong tool

- **You want to ship the fix for one specific PR's review.** Go straight to `/oss-contribute:contribute-upstream <pr-url>` — Phase 8 is the whole point.
- **You want a portfolio artifact from merged PRs.** `/oss-contribute:log`.
- **You want something new to work on.** `/oss-contribute:find-issues`.
- **You want a one-off "is this PR green?" check.** `gh pr checks <url>` is one line.
