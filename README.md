# oss-contribute

A Claude Code plugin that turns the bugs you hit in third-party dependencies into upstream contribution PRs — **without leaving the project you're working on**.

Most "contribute to OSS" tools assume you're shopping for something to work on. This one is built for the other case: you're shipping a feature in your own repo, you hit a bug in a library you depend on, and you want to fix it upstream and unblock yourself in the same session. The reactive path is the focus; the proactive path is the sibling.

## Why this instead of [other plugin]

There are a couple of Claude Code plugins in this space already. Honest comparison:

| | [LuciferDono/contribute](https://github.com/LuciferDono/contribute) | [mainnebula/token-steward](https://github.com/mainnebula/Token-Steward) | **oss-contribute** |
|---|---|---|---|
| Trigger style | Proactive only | Proactive only | **Both proactive + reactive** |
| Invoked from consumer repo | ✗ | ✗ | ✓ (auto-detect upstream from `package.json`) |
| Local-patch handoff during PR review | ✗ | ✗ | ✓ (`pnpm patch` / fork-branch overrides) |
| Token-based duplicate-PR search | ✗ | ✗ | ✓ (`%5F` literal-token case caught) |
| Invitation-only repo HARD STOP | ✗ | ✗ | ✓ (saves wasted clones on e.g. `openai/codex`) |
| Pre-PR confirmation gate | rule (Rule 1) | not explicit | ✓ baked step (Phase 5 step 7) |
| Tractability gate (catch half-fixes / re-implementation) | ✗ | ✗ | ✓ (Phase 3 BLOCKING) |
| Propose-instead-of-PR escape | ✗ | ✓ | ✓ (ported from token-steward) |
| Respond-to-review phase | ✓ | ✗ | ✓ (ported from LuciferDono, stricter rules) |
| License | BSL-1.1 (non-OSI) | MIT | MIT |

The differentiator isn't volume of features — it's that **oss-contribute is invoked from inside the project you broke against**, and stays useful all the way from "I hit a bug" through "PR shipped and my project is unblocked while it's in review."

## What's inside

| Skill | Trigger | Job |
|---|---|---|
| `find-issues` | `/oss-contribute:find-issues` | **Proactive** — ranked shortlist of ripe issues across your watched repos. Read-only; hands off to `contribute-upstream`. |
| `contribute-upstream` | `/oss-contribute:contribute-upstream <pkg>` | **Reactive** — 8 phases from "I hit a bug in this package" to a merged PR. Pre-flight gate, repro bridge, fix, PR with confirmation gate, local-patch handoff, review response. |
| `profile` | `/oss-contribute:profile` | View/edit the shared preferences file used by both skills. |

All three skills read from one shared profile so settings never drift.

## Install

> **Not yet on the official Anthropic marketplace.** Install directly from this repo:

```
/plugin marketplace add shiminshen/oss-contribute
/plugin install oss-contribute@oss-contribute
```

After install, set up your profile:

```
/oss-contribute:profile edit
```

## The reactive flow (killer feature)

You're working in your project. You hit a bug in `better-auth`. From inside your project:

```
/oss-contribute:contribute-upstream better-auth
```

The skill walks through 8 phases:

| Phase | What |
|---|---|
| **1 — Pre-flight gate (BLOCKING)** | Resolve upstream from `package.json`. Read `CONTRIBUTING.md` / `docs/contributing.md` / `.github/pull_request_template.md` (every variant). HARD STOP for invitation-only repos. Duplicate-PR search by tokens from the issue body (not just title keywords). CLA check. Pick GitHub account (asks every time when multiple are logged in). Confirm with user. |
| **2 — Setup** | Re-verify duplicate immediately before clone. Fork → sibling-directory clone → install deps with project's pinned package manager → baseline tests pass. |
| **3 — Repro bridge** | Translate the consumer-side symptom into a failing test in the upstream's own test framework. **Tractability gate (BLOCKING)** — catches re-implementation framed as bug fix, type-only patches that hide runtime bugs. Bail to Phase 6 if not session-sized. |
| **4 — Fix** | Smallest possible diff. Regression marker if the project uses one. |
| **5 — PR prep** | Branch name, commit format, changeset, branch target — all matched to the project's conventions. **Pre-PR confirmation gate (BLOCKING)** before `gh pr create`. |
| **6 — Escape hatches** | **6a:** file an issue with consumer-side repro when no upstream repro is feasible. **6b: Propose** — post a structured proposal comment on an existing issue when the fix is too large for one session. |
| **7 — Local-patch handoff** | `pnpm patch` or `pnpm.overrides` → your fork branch, so your project isn't blocked while the PR sits in review. |
| **8 — Respond to PR review** | Fetch review state, classify each feedback item (apply / push back / clarify / out-of-scope), apply only after user confirmation, reply on every thread, never silently re-push. |

## The proactive flow (sibling)

```
/oss-contribute:find-issues
/oss-contribute:find-issues --lang typescript --budget 1h
```

Searches your watched repos for issues with `good first issue` / `help wanted` / `bug` labels (per-repo subagents to keep raw JSON out of the main context), drops anything already claimed (assignee, linked PR, "we're working on this" comments), drops anything mid-rewrite where the "fix" would be a re-implementation, drops anything in invitation-only repos, and ranks the rest by repro quality, scope, stack match, and repo health. Hands off to `contribute-upstream` once you pick one — no auto-invoke.

## Profile

Lives at one of:

- `$CLAUDE_PLUGIN_DATA/profile.md` when installed as a plugin (survives plugin updates)
- `~/.claude/plugins/data/oss-contribute/profile.md`
- `~/.claude/skills/oss-contribute/profile.md` (local-development mode)

See [`docs/profile.example.md`](./docs/profile.example.md) for the schema.

## Hard rules baked in

- **Never** push to upstream main. Always work from your fork.
- **Never** open a PR without showing the full body and getting your explicit confirmation.
- **Never** assume which GitHub account to use. If multiple `gh` accounts are logged in, asks every time.
- **No AI-attribution trailers on commits.** No `Co-Authored-By: Claude` / `Generated-By` / "AI assisted" markers — anywhere. The contribution reads as sole-authored by you.
- **Surface, don't suppress.** If a contribution probably won't be accepted (dormant repo, scope mismatch, CLA blocker, duplicate PR, invitation-only, re-implementation disguised as bug fix), the skill says so before doing the work.
- Honors each project's AI-contribution policy. If `CONTRIBUTING.md` requires an issue before a feature PR, the skill files or finds one first.

## Receipts

Real contributions shipped using this workflow:

<!-- - [`<owner>/<repo>#<n>`](https://github.com/.../pull/N) — short description (YYYY-MM-DD) -->

_(none yet — first receipt expected to be [better-auth#9605](https://github.com/better-auth/better-auth/pull/9605) once it merges)_

## Versioning & changelog

[Semantic versioning](https://semver.org). See [CHANGELOG.md](./CHANGELOG.md) for the full history. Notable v0.x decisions:

- **v0.1** — three skills + shared profile
- **v0.2** — preflight hardening (subagent dispatch, invitation-only HARD STOP, token-based dup-PR search, tractability gate)
- **v0.3** — Propose escape hatch, Phase 8 (review response), no-AI-attribution rule

## Roadmap

- `pipeline` skill — status of your open/merged PRs across forks
- `log` skill — portfolio entry generated from merged contributions
- `do` / `guide` / `adaptive` operating modes
- Per-repo conventions cache

## Contributing

Issues and PRs welcome. For new skills, open an issue first to align on scope. For fixes, a failing test + minimal diff is the path of least resistance. (The plugin is literally the workflow we want you to use.)

## License

MIT — see [LICENSE](./LICENSE).
