---
name: contribute-upstream
description: Turn a bug you hit in a third-party dependency (while working in your own project) into an upstream contribution PR — or a well-filed issue if a clean upstream repro isn't feasible. Handles the boring parts — reading CONTRIBUTING, duplicate search, package-manager setup, commit/PR conventions, changesets — and a local patch handoff so the consumer repo isn't blocked.
trigger: /oss-contribute:contribute-upstream
---

# contribute-upstream

Invoke from inside the **consumer** repo (the one that depends on the buggy package), not the upstream itself. You do the boilerplate the user shouldn't have to memorise per-repo; the user provides the bug context.

## Usage

```
/oss-contribute:contribute-upstream <package-name>          # e.g. better-auth
/oss-contribute:contribute-upstream <package-name> <symptom-summary>
/oss-contribute:contribute-upstream                         # infer package from current stack trace / open file
```

## Profile location

Read the profile in this order (used only for default GitHub account):

1. `$CLAUDE_PLUGIN_DATA/profile.md`
2. `~/.claude/plugins/data/oss-contribute/profile.md`
3. `~/.claude/skills/oss-contribute/profile.md`

Profile is **optional** for this skill — the GitHub-account question is asked interactively either way (see Phase 1 step 7).

## Phase 1 — Pre-flight gate (BLOCKING)

Do all of the following before touching any code. If any step surfaces a blocker, stop and tell the user; do not proceed unilaterally.

0. **Freshness re-check (FIRST, BLOCKING).** Before *any* other Phase 1 work — before resolving the upstream repo, before dispatching the rules-of-the-road subagent, before reading anything — re-verify the issue is still ripe RIGHT NOW. Even if you just hand-picked it from `find-issues` Phase 4 minutes ago. State changes fast on Hot repos.

   Single batched call:
   - `gh issue view <n> --repo <owner>/<repo> --json assignees,closedByPullRequestsReferences,state`
   - `gh search prs --repo <owner>/<repo> --state open --limit 5 "#<n>"`
   - One or two distinctive backticked identifiers from the issue body

   Drop and surface to the user immediately if **any** of these is true:
   - Issue now has an assignee (someone is on it — competing is rude)
   - `closedByPullRequestsReferences` is non-empty (a PR is already linked)
   - A token-search PR hits the same fix surface
   - Issue state is no longer `OPEN`

   Motivating case: `mastra-ai/mastra#16422` (late assignee). See `references/case-studies.md#freshness--late-assignee-phase-1-step-0`.

0a. **Adjacent-stalled-PR check (BLOCKING).** Search for any open PR in the *same code area* that has been stalled. Run `gh pr view <stalled-pr-number> --json reviews,comments,reviewDecision` — **zero reviews + zero comments + `REVIEW_REQUIRED` for 30+ days** is the "dead area" signal. Surface to the user before investing.

   Motivating case: `vercel/ai#13962` adjacent to stalled `#12924`. See `references/case-studies.md#adjacent-stalled-pr--dead-area-signal-phase-1-step-0a`.

0b. **Already-fixed-on-main check (BLOCKING).** Read the version the reporter is on from the issue body. Compare to current `main`'s version + recent commits to the file path the reporter mentions. If a fix-shaped commit landed *between the reporter's version and main*, the bug may already be fixed — the reporter just needs to upgrade. Verify by running the existing tests for that file. If they pass on main, drop and offer to comment pointing the reporter to the version that fixes it.

   Cheap check:
   ```
   gh api 'repos/<owner>/<repo>/commits?path=<file>&per_page=15' \
     --jq '.[] | {sha: .sha[:8], date: .commit.author.date, msg: .commit.message | split("\n")[0]}'
   ```

   Motivating cases: `assistant-ui#4009`, `mastra#16383`, and the inverse `drizzle-orm#5755`. See `references/case-studies.md#already-fixed-on-main--wrong-slice-of-the-version-axis-phase-1-step-0b`.

1. **Resolve the upstream repo.** From the consumer's `package.json` + lockfile, get the installed version and the `repository` URL. Disambiguate (workspace, fork, mirror) with the user if needed.
2. **Read the rules of the road.** **Strongly prefer dispatching this to a `general-purpose` subagent** — many file fetches, mostly null results, content dumps that pollute the main context. The subagent returns a compact verdict (≤15 lines) covering: contribution policy (open / discuss-first / invitation-only — HARD STOP if invitation-only), signed-commits requirement, CLA, branch target, changeset usage, packageManager + node version. Pass the upstream `owner/repo` and the issue number for context.

   Fetch and read each of these paths; treat the **first hit** as authoritative for that doc type. Filenames vary in case and folder — try every variant before giving up:
   - **Contributing policy**: `CONTRIBUTING.md`, `.github/CONTRIBUTING.md`, `docs/CONTRIBUTING.md`, `docs/contributing.md`, `docs/contribute.md`. **Many projects put the real policy at `docs/contributing.md` while the root file is missing or stub** — keep searching after the first 404.
   - `CODE_OF_CONDUCT.md`
   - `SECURITY.md` (also try `.github/SECURITY.md`) — if the bug is a security issue, **STOP**: most projects forbid public disclosure. Redirect to the project's security contact.
   - **PR template**: `.github/PULL_REQUEST_TEMPLATE.md` AND `.github/pull_request_template.md` (case matters on GitHub's API). Also `PULL_REQUEST_TEMPLATE.md` at root. **The template often restates policy** ("invitation only", "must link issue", "do not open without issue first") — read it as policy, not just formatting.
   - `CLAUDE.md` / `AGENTS.md` at repo root (project-specific AI guidance)
   - `.changeset/config.json` (does the project use changesets?)
   - `package.json#packageManager` and `.nvmrc`

   **Invitation-only / closed-contribution check (HARD STOP).** Some projects (e.g. `openai/codex`) accept external PRs **by invitation only** and close uninvited PRs unread. Scan the contributing doc and PR template for phrases like "invitation only", "do not accept unsolicited", "closed without review", "external contributions are closed". If found, STOP — do not proceed to Phase 2 clone. Instead, switch to Phase 6 (issue-only path) and offer to comment on the issue with analysis + suggested fix, which is what these projects explicitly invite. Surface this clearly to the user before any further work.
3. **Check signs of life.** Recent merged PRs (last 30d), issue response cadence, last release. If the project looks dormant or hostile to outside PRs, surface that and ask whether to continue.
4. **Check whether discussion is required.** Some projects explicitly require an issue before a feature PR and will close cold feature PRs. Bug fixes are usually fine. For features, file or find the issue first.
5. **Duplicate search.** Title-keyword searches miss PRs whose title describes the *implementation* rather than the *symptom*. Search by tokens extracted from the issue body.

   **Dispatch all four token queries in parallel — ONE message with one Bash tool call per token type.** Do not run them sequentially. Inspect results together; if any returns a PR hit, short-circuit and drop.

   The token types:

   1. **The issue number itself.** `gh search prs --repo <owner>/<repo> --state open --limit 5 "<n>"` — many PR descriptions reference the issue.
   2. **URL-encoded or other distinctive literals** in the issue body — `%5F`, error codes, magic strings.
   3. **Backticked code identifiers** from the issue body — function names, file paths, type names.
   4. **Error message fragments** quoted in the body, if any.

   Title-keyword paraphrases are a last resort, not a first resort. Motivating case: the `%5F` literal-token search. See `references/case-studies.md#token-based-duplicate-pr-search--implementation-titled-prs-phase-1-step-5`.

   Also run `gh issue view <n> --json assignees,closedByPullRequestsReferences,comments` on every candidate issue. Outcomes:
   - Open issue, no assignee, no linked PR → add the user's extra context as a comment; ask in the same comment whether you can take it (some projects require an explicit "assign me" / `/assign` before you start).
   - Open issue **with an assignee** but no PR yet → comment asking if they're still actively working on it before duplicating effort. Do not start work until you hear back or the assignee is removed.
   - Open issue **with a linked open PR** → tell the user; offer to review/test that PR, not compete with it.
   - Open PR found via `gh search prs` for the same fix → same: review/test, don't compete.
   - Closed PR → read why; that reason (scope, design objection, maintainer pushback) often blocks the same fix even if the bug is real.
6. **CLA check.** Look for `.github/cla.yml`, `cla-assistant`, or a CLA note in CONTRIBUTING. Surface CLA requirements before any code work.
7. **Pick the GitHub account AND the git commit identity (must match).** Read the shared profile's `## Default GitHub account` AND `## Git commit identity` (the `name:` and `email:` lines). **Use both without re-prompting** — even when `gh auth status` shows multiple accounts logged in. The profile entries are the user's standing instructions; re-asking on every contribution treats stable preferences as ephemeral and creates friction. State the chosen account + name + email once in the Phase 1 step 8 summary so they're visible, but do not put them behind questions. The user will override per-invocation if they want a different identity ("use the company account this time"); otherwise honour the profile.

   **Critical: commit identity must match GitHub account.** The default `git config --global user.email` is typically the user's day-job email (e.g. `damon@deeplearning.ai`). If the OSS clone inherits this, the PR appears under the personal GitHub account BUT every commit is stamped with the company email — exactly the wrong signal, and visible forever in `git log`. After the Phase 2 clone, set local repo identity explicitly:

   ```
   git -C <clone-dir> config user.name "<profile name>"
   git -C <clone-dir> config user.email "<profile email>"
   ```

   Verify before any commit:
   ```
   git -C <clone-dir> config user.email
   # must match the profile's email, not the global one
   ```

   If the profile has no `## Git commit identity` section, fall back to asking the user explicitly — never silently inherit the global identity for an OSS clone. **Only re-prompt for account/identity** if the profile fields are absent, or if the user has corrected the choice within the same session.
8. **Confirm with user.** Summarise findings (repo, version, dup status, conventions, CLA, branch target, changeset y/n, chosen GitHub account, chosen git identity) in ≤10 lines. Get explicit go-ahead before Phase 2.

## Phase 2 — Setup

0. **Re-verify duplicate (BLOCKING).** Re-run the duplicate-PR search from Phase 1 step 5, *right now*, immediately before the clone. Hunt → contribute can take minutes to days; PRs land in that window. Cloning a large monorepo costs 15–25 minutes — confirm it's still worth doing.

   If a new PR has appeared, stop and offer to review/test it instead of competing.

1. Clone the user's **fork** (create one with `gh repo fork` first if absent) to a **sibling directory** of the consumer repo — never inside it.
2. **Set local git identity (BLOCKING before any commit).** Immediately after clone, set the repo-local `user.name` and `user.email` from the profile's `## Git commit identity` so commits are not stamped with the user's global (typically work) identity:
   ```
   git -C <clone-dir> config user.name "<profile name>"
   git -C <clone-dir> config user.email "<profile email>"
   git -C <clone-dir> config user.email   # verify
   ```
   This is the safety net for the Phase 1 step 7 decision. If the profile lacks the identity section, ask the user explicitly — never let the global identity be inherited by an OSS clone. Documented failure mode: PR appears from personal GitHub account but `git log` shows every commit under the work email, visible forever.
3. Install deps with the project's pinned package manager (`corepack prepare` / `corepack enable` if needed). Honour `.nvmrc`.
4. Run the project's baseline `test` and `typecheck` to confirm a clean starting state. If they fail on the default branch, surface that — do not try to "fix" baseline failures.

## Phase 3 — Reproduce in the upstream's own test framework

This is the hardest step. In order:

0. **Convention scan (BLOCKING before any code, including the failing test).** Before writing a single line — even the failing test — load `references/convention-checklist.md` and run it against the file you'll modify and its closest test file. Capture the test-file structure, helper / mock-factory patterns, naming conventions, cross-cutting setup placement, comment density, lifecycle / async idioms, and import style. Also read recent merged PRs touching adjacent files (`gh pr list --search "<file/path>" --state merged --limit 3`) — they show what conventions the maintainers *enforce in review*. Prevention counterpart to Phase 4's audit; skipping it means the audit catches mistakes after they're written and produces review churn.

1. **Adapt the upstream's existing tests.** Find the test file closest to the affected code and add a failing case that mirrors the consumer-side symptom — using the conventions captured in step 0.
2. **Use the project's test harness** — do not invent a custom setup.
3. **Bail out** if the bug depends on consumer-stack specifics the upstream can't reproduce (specific framework version, DB driver, env-specific behaviour). Switch to Phase 6 (issue-only).
4. **Tractability gate (BLOCKING).** Before writing the fix, ask: *is the fix scope what the issue framing suggested?* Two failure modes to catch:
   - **Feature gone, not bug.** The issue says "X is missing/broken in version Y" but X is wholly absent from the new code paths. The "fix" would be a re-implementation, not a bug fix — switch to Phase 6 with analysis: "feature X is absent — intentional or oversight?" Motivating case: `drizzle-orm#5755`. See `references/case-studies.md#tractability-gate--feature-gone-not-bug-phase-3-step-4`.
   - **Half-fix risk.** The minimal fix (e.g. one type addition) makes TS stop erroring but leaves runtime semantics broken. Don't ship half-fixes — they hide the bug from users. Either fix both or switch to Phase 6.

Confirm the new test fails for the **right** reason before writing a fix.

## Phase 4 — Fix

- Smallest possible diff. Bug fix only — no surrounding refactor, no scope creep, no comments restating the obvious.
- Run the targeted test → the full affected file → the project's `typecheck`.
- If the fix touches a public API, update `docs/` per the project's docs convention.
- Add a regression marker if the project uses one (e.g. `@see https://github.com/.../issues/<n>` block above the test).
- **Convention audit (BLOCKING before commit).** Re-load `references/convention-checklist.md` and run it against your diff. Verification counterpart to Phase 3 step 0's prevention scan — catch what slipped through. Motivating case: CopilotKit#4798. See `references/case-studies.md#convention-divergence-at-commit-phase-3-step-0--phase-4-audit`.

## Phase 5 — PR prep

1. **Branch name.** Match the project's pattern (`fix/<slug>`, `feat/<slug>`, etc.).
2. **Commit format.** Strict match to CONTRIBUTING (Conventional Commits, lowercase subject, scope, `!` for breaking, etc.).
3. **Changeset.** If the project uses changesets, write one. Style rules:
   - Written for end users reading the changelog, not the reviewer.
   - Describe the symptom users see, not the internal cause.
   - No commit-style prefix.
   - Pick bump type by user impact (patch / minor / major).
4. **Branch target.** Read CONTRIBUTING for `main` vs `next` vs `develop` vs `master`.
5. **PR body.** Follow the project's PR template. Always include: Summary · Closes #<issue> · Test plan · Breaking changes (if any).
6. **Author identity.** Use the GitHub account and git identity the user confirmed in Phase 1 step 7. Do not silently fall back to the active `gh` account.
7. **Pre-PR confirmation gate (BLOCKING).** Before calling `gh pr create`, show the user:
   - Target repo and base branch (`upstream/owner:base`)
   - Head ref (`<user>:<branch>`)
   - PR title
   - Full PR body
   - List of commits that will be included
   - List of files changed (summary, not full diff)

   Ask for explicit confirmation. Do not open the PR until the user says yes. If they ask for edits, revise and re-show — never push or open the PR on your own initiative.
8. **Push from fork, open PR cross-repo** only after step 7 confirmation: `gh pr create --repo upstream/repo --head user:branch`.

## Phase 6 — Issue-only or Propose escape hatch

When Phase 3 (repro bridge) or the Phase 3 tractability gate blocks the PR path, **do not** abandon the contribution — switch to the most useful artifact you can still produce. Pick one of two outputs:

### 6a — File a new issue

Use when no upstream issue exists yet (you discovered the bug from your consumer-side symptom).

- Use the project's bug-report template.
- Include: installed version, exact symptom, minimal consumer-side repro, expected vs actual, environment.
- Link related closed issues/PRs found in Phase 1.
- Do **not** open a PR in this path.

### 6b — Post a Proposal comment

Use when an upstream issue already exists but the fix is too large, too design-sensitive, or otherwise not session-sized. Maps to what Token-Steward calls the "Propose" action: a structured comment that gauges maintainer interest before anyone writes code.

Structure the comment as:

1. **Problem restated in one line** — confirms you understood the report.
2. **Root-cause analysis** — what's broken in the codebase, with file:line references where useful.
3. **Proposed approach** — the design sketch, not the diff. 3–6 bullets max.
4. **Open questions for the maintainer** — explicit asks: "Is this the right surface to fix?", "Should this be a breaking change?", "Is there a related refactor in flight?"
5. **What I'd need to ship it** — assignment, API blessing, test-strategy guidance.

**Hard rules for proposal comments:**

- The comment is read-only output to the user first; do not post until the user confirms.
- Use the issue's existing thread; do not open a duplicate issue.
- Do **not** include the diff. Description sketches, not implementations — let the maintainer steer before code is written.

## Phase 7 — Local patch handoff (consumer side)

The upstream PR may sit in review for days or weeks. Don't leave the consumer blocked:

- **`pnpm patch <pkg>`** → reapply the same fix as a local patch; commit `patches/*.patch` to the consumer repo.
- **`pnpm.overrides`** → point the consumer at the user's fork branch (`github:<user>/<repo>#<branch>`).
- **npm/yarn equivalents** if the consumer uses those.

Leave a TODO in the consumer repo noting the upstream PR number so the patch can be removed once the fix is released.

## Phase 8 — Respond to PR review

After the PR is open, this phase handles the round-trip until merge or close.

**Trigger** when `gh pr view <n> --json reviews,comments,reviewRequests` shows new activity since last check, or the user invokes `/oss-contribute:contribute-upstream` with the PR URL/number after it's open.

**Procedure:** Load `references/phase-8-review.md`. It contains the 7-step procedure (fetch → classify into apply/push-back/clarify/out-of-scope → summarise to user → apply approved → push to fork → reply on threads → re-request review) and 4 hard rules (no silent re-pushing, no mid-review rebase, no force-push unless asked, escalate when feedback conflicts with Phase 1 hard rules).

## Hard rules

- **Never** push to the upstream's main repo. Always work from the user's fork.
- **Never** commit without explicit user instruction.
- **Never** amend or force-push a published commit unless the user explicitly asks.
- **Never** open a PR for a new feature or breaking change without an existing or freshly-filed issue, if the project's CONTRIBUTING requires one.
- **No AI-attribution trailers on commits.** Never add `Co-Authored-By: Claude` / `Generated-By` / any "AI assisted this commit" line. OSS maintainers treat these as noise at best, hostile at worst — the contribution must read as sole-authored by the GitHub user. Determine identity via `gh api user --jq '.login'` and `git config user.name` / `git config user.email`. Zero exceptions, including in commit-message bodies and PR bodies.
- **Surface, don't suppress.** If any phase reveals the contribution probably won't be accepted (dormant repo, scope mismatch, CLA blocker, duplicate open PR, scope-creep risk), say so before doing the work.

## Output shape

At the end, produce:

- The upstream PR URL (Phase 5), issue URL (Phase 6a), or proposal-comment permalink (Phase 6b).
- The consumer-side patch/override instructions actually applied (if Phase 7).
- The post-review state if Phase 8 ran: open feedback threads remaining, last push SHA, current `reviewDecision`.
- A one-line tracker note: package, version, PR/issue #, what to remove once released.
