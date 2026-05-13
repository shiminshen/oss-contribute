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

1. **Resolve the upstream repo.** From the consumer's `package.json` + lockfile, get the installed version and the `repository` URL. Disambiguate (workspace, fork, mirror) with the user if needed.
2. **Read the rules of the road.** Fetch and read:
   - `CONTRIBUTING.md` (or `.github/CONTRIBUTING.md`)
   - `CODE_OF_CONDUCT.md`
   - `SECURITY.md` — if the bug is a security issue, **STOP**: most projects forbid public disclosure. Redirect to the project's security contact.
   - `.github/PULL_REQUEST_TEMPLATE.md`
   - `CLAUDE.md` / `AGENTS.md` at repo root (project-specific AI guidance)
   - `.changeset/config.json` (does the project use changesets?)
   - `package.json#packageManager` and `.nvmrc`
3. **Check signs of life.** Recent merged PRs (last 30d), issue response cadence, last release. If the project looks dormant or hostile to outside PRs, surface that and ask whether to continue.
4. **Check whether discussion is required.** Some projects explicitly require an issue before a feature PR and will close cold feature PRs. Bug fixes are usually fine. For features, file or find the issue first.
5. **Duplicate search.** Title-keyword searches miss PRs whose title describes the *implementation* rather than the *symptom*. Search by tokens extracted from the issue body, in this order; stop at the first PR hit:

   1. **The issue number itself.** `gh search prs --repo <owner>/<repo> "<n>"` — many PR descriptions reference the issue.
   2. **URL-encoded or other distinctive literals** in the issue body — `%5F`, error codes, magic strings.
   3. **Backticked code identifiers** from the issue body — function names, file paths, type names.
   4. **Error message fragments** quoted in the body, if any.
   5. Title-keyword paraphrases as a last resort, not a first resort.

   Documented failure case: issue titled "Layouts for paths that start with underscore (%5F)…" had an open PR titled "fix(typegen): normalize %5F to _…" — caught only by the `%5F` literal-token search, not by title-keyword variants.

   Also run `gh issue view <n> --json assignees,closedByPullRequestsReferences,comments` on every candidate issue. Outcomes:
   - Open issue, no assignee, no linked PR → add the user's extra context as a comment; ask in the same comment whether you can take it (some projects require an explicit "assign me" / `/assign` before you start).
   - Open issue **with an assignee** but no PR yet → comment asking if they're still actively working on it before duplicating effort. Do not start work until you hear back or the assignee is removed.
   - Open issue **with a linked open PR** → tell the user; offer to review/test that PR, not compete with it.
   - Open PR found via `gh search prs` for the same fix → same: review/test, don't compete.
   - Closed PR → read why; that reason (scope, design objection, maintainer pushback) often blocks the same fix even if the bug is real.
6. **CLA check.** Look for `.github/cla.yml`, `cla-assistant`, or a CLA note in CONTRIBUTING. Surface CLA requirements before any code work.
7. **Pick the GitHub account.** If the shared profile exists, read its `## Default GitHub account` as the proposed default. Run `gh auth status` to list logged-in accounts. If more than one is present, **ask the user explicitly** which account to fork from and open the PR with — do not assume the active `gh` account, and do not silently use the profile default. Also confirm the git author identity (`user.name`, `user.email`) to use for the commit; default to the user's **personal** identity unless they say otherwise.
8. **Confirm with user.** Summarise findings (repo, version, dup status, conventions, CLA, branch target, changeset y/n, chosen GitHub account, chosen git identity) in ≤10 lines. Get explicit go-ahead before Phase 2.

## Phase 2 — Setup

0. **Re-verify duplicate (BLOCKING).** Re-run the duplicate-PR search from Phase 1 step 5, *right now*, immediately before the clone. Hunt → contribute can take minutes to days; PRs land in that window. Cloning a large monorepo costs 15–25 minutes — confirm it's still worth doing.

   If a new PR has appeared, stop and offer to review/test it instead of competing.

1. Clone the user's **fork** (create one with `gh repo fork` first if absent) to a **sibling directory** of the consumer repo — never inside it.
2. Install deps with the project's pinned package manager (`corepack prepare` / `corepack enable` if needed). Honour `.nvmrc`.
3. Run the project's baseline `test` and `typecheck` to confirm a clean starting state. If they fail on the default branch, surface that — do not try to "fix" baseline failures.

## Phase 3 — Reproduce in the upstream's own test framework

This is the hardest step. In order:

1. **Adapt the upstream's existing tests.** Find the test file closest to the affected code and add a failing case that mirrors the consumer-side symptom.
2. **Use the project's test harness** — do not invent a custom setup.
3. **Bail out** if the bug depends on consumer-stack specifics the upstream can't reproduce (specific framework version, DB driver, env-specific behaviour). Switch to Phase 6 (issue-only).

Confirm the new test fails for the **right** reason before writing a fix.

## Phase 4 — Fix

- Smallest possible diff. Bug fix only — no surrounding refactor, no scope creep, no comments restating the obvious.
- Run the targeted test → the full affected file → the project's `typecheck`.
- If the fix touches a public API, update `docs/` per the project's docs convention.
- Add a regression marker if the project uses one (e.g. `@see https://github.com/.../issues/<n>` block above the test).

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

## Phase 6 — Issue-only escape hatch

If Phase 3 can't produce a minimal upstream repro, switch to filing an issue:

- Use the project's bug-report template.
- Include: installed version, exact symptom, minimal consumer-side repro, expected vs actual, environment.
- Link related closed issues/PRs found in Phase 1.
- Do **not** open a PR in this path.

## Phase 7 — Local patch handoff (consumer side)

The upstream PR may sit in review for days or weeks. Don't leave the consumer blocked:

- **`pnpm patch <pkg>`** → reapply the same fix as a local patch; commit `patches/*.patch` to the consumer repo.
- **`pnpm.overrides`** → point the consumer at the user's fork branch (`github:<user>/<repo>#<branch>`).
- **npm/yarn equivalents** if the consumer uses those.

Leave a TODO in the consumer repo noting the upstream PR number so the patch can be removed once the fix is released.

## Hard rules

- **Never** push to the upstream's main repo. Always work from the user's fork.
- **Never** commit without explicit user instruction.
- **Never** amend or force-push a published commit unless the user explicitly asks.
- **Never** open a PR for a new feature or breaking change without an existing or freshly-filed issue, if the project's CONTRIBUTING requires one.
- **Surface, don't suppress.** If any phase reveals the contribution probably won't be accepted (dormant repo, scope mismatch, CLA blocker, duplicate open PR, scope-creep risk), say so before doing the work.

## Output shape

At the end, produce:

- The upstream PR URL (or upstream issue URL, if Phase 6).
- The consumer-side patch/override instructions actually applied (if Phase 7).
- A one-line tracker note: package, version, PR/issue #, what to remove once released.
