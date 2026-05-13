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

For each repo in the (filtered) watched list, in parallel:

```
gh search issues \
  --repo <owner>/<repo> \
  --state open \
  --label "good first issue" \
  --label "help wanted" \
  --label "bug" \
  --sort updated \
  --limit 30
```

`gh search issues --label` is AND, not OR — run one query per relevant label and combine result sets in the skill. Apply args filters (language, repo, budget) and profile filters.

Cap the total candidate pool at ~60 across all repos before triage, to keep Phase 3 cheap.

## Phase 3 — Triage each candidate

For every survivor, fetch:

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

### Duplicate-PR search

Issue-title keywords miss PRs whose title describes the *implementation* rather than the *symptom*. Search by tokens extracted from the issue body, not by paraphrases of the title.

For each candidate, run `gh search prs --repo <owner>/<repo> --state open <token>` against each of these, in order, and stop at the first hit:

1. **The issue number itself.** `"#93700"` and bare `"93700"` — many PR descriptions reference it.
2. **URL-encoded or other distinctive literals** in the issue body — `%5F`, error codes, magic strings.
3. **Backticked code identifiers** from the issue body — function names, file paths, type names (e.g. `` `LayoutRoutes` ``, `` `buildUpdateSet` ``).
4. **Error message fragments** quoted in the body, if any.

A title-keyword search alone is not enough. Documented failure case: issue titled "Layouts for paths that start with underscore (%5F)…" had an open PR titled "fix(typegen): normalize %5F to _…" — caught only by the `%5F` token search.

If a PR hit appears, drop the candidate.

For each survivor, score on:

- **Repro quality** — does the body have steps + expected vs actual?
- **Scope estimate** — `one-liner` / `small` / `medium` / `large`. Drop `large` from a "weekend"-budget shortlist.
- **Stack match** — overlap with the user's languages/frameworks.
- **Repo health** — recent merge cadence, last release date.
- **Maintainer signal** — has a maintainer triaged/labelled it?

## Phase 4 — Output

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
