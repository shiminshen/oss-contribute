# oss-contribute

A Claude Code plugin that turns the bugs you hit in third-party dependencies into upstream contribution PRs — **without leaving the project you're working on**.

Most "contribute to OSS" tools assume you're shopping for something to work on. This one is built for the other case: you're shipping a feature in your own repo, you hit a bug in a library you depend on, and you want to fix it upstream and unblock yourself in the same session. The reactive path is the focus; the proactive path is the sibling.

## What's inside

| Skill | Trigger | Job |
|---|---|---|
| `find-issues` | `/oss-contribute:find-issues` | **Proactive** — ranked shortlist of ripe issues across your watched repos. Read-only; hands off to `contribute-upstream`. |
| `contribute-upstream` | `/oss-contribute:contribute-upstream <pkg>` | **Reactive** — 8 phases from "I hit a bug in this package" to a merged PR. Pre-flight gate, repro bridge, fix, PR with confirmation gate, local-patch handoff, review response. |
| `log` | `/oss-contribute:log` | **Portfolio** — render your merged upstream PRs (default last 90 days) as a hybrid table + detail-block artifact. Read-only; writes a local file or stdout. |
| `profile` | `/oss-contribute:profile` | View/edit the shared preferences file used by every skill. |

All skills read from one shared profile so settings never drift.

## Install

> **Not yet on the official Anthropic marketplace.** Clone locally and install from the path.

```bash
# 1. Clone the repo anywhere on your machine
git clone https://github.com/shiminshen/oss-contribute.git
cd oss-contribute
```

Then in Claude Code (use the absolute path to your clone):

```
/plugin marketplace add /absolute/path/to/oss-contribute
/plugin install oss-contribute@oss-contribute
```

No `npm install` or build step — the plugin is plain markdown.

### Verify the install

```
/oss-contribute:profile
```

Expected output: `No profile found. … Run: /oss-contribute:profile edit to set one up.` That message means the three skills loaded correctly under the `oss-contribute:` namespace.

### First-run setup

```
/oss-contribute:profile edit
```

Walks you through:

- Which GitHub repos to watch (e.g. `better-auth/better-auth`, `vercel/next.js`)
- Languages and stack you work in
- Which `gh`-logged-in account to fork/PR from (asks every time when multiple are present, but seeds a sensible default)
- Default time budget (`30m` / `1h` / `weekend`)
- What "ripe" means to you (heuristic seed for `find-issues` ranking)

Profile is written to `$CLAUDE_PLUGIN_DATA/profile.md` when installed as a plugin (survives plugin updates), or `~/.claude/skills/oss-contribute/profile.md` in local-dev mode.

### Update later

```bash
cd /path/to/oss-contribute
git pull
```

Then in Claude Code: `/plugin reload oss-contribute` (or uninstall + reinstall).

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

## Checking your pipeline

No skill for this — there's no value beyond what `gh` already gives you. The one-liner:

```bash
# Open PRs, with review state and last activity
gh search prs --author @me --state open \
  --json url,title,repository,reviewDecision,updatedAt \
  --template '{{range .}}{{.repository.nameWithOwner}}#{{.number}}  {{.reviewDecision}}  {{timeago .updatedAt}}  {{.title}}{{"\n"}}{{end}}'

# Merged in last 90 days (receipts)
gh search prs --author @me --state merged --merged ">=$(date -v-90d +%Y-%m-%d)"
```

`reviewDecision: CHANGES_REQUESTED` is the signal to re-enter `contribute-upstream` Phase 8 on that PR. If you find yourself running this often enough to want a skill, that's the signal to revisit the Roadmap below — until then, GitHub notifications + this command are enough.

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

- [`cloudflare/workers-sdk#13908`](https://github.com/cloudflare/workers-sdk/pull/13908) — `fix(wrangler): stop rewriting query strings that contain the request Host` (merged 2026-05-15, +14/-1 across 2 files)

<!-- - [`<owner>/<repo>#<n>`](https://github.com/.../pull/N) — short description (YYYY-MM-DD) -->

## Versioning & changelog

[Semantic versioning](https://semver.org). See [CHANGELOG.md](./CHANGELOG.md) for the full history. Notable v0.x decisions:

- **v0.1** — three skills + shared profile
- **v0.2** — preflight hardening (subagent dispatch, invitation-only HARD STOP, token-based dup-PR search, tractability gate)
- **v0.3** — Propose escape hatch, Phase 8 (review response), no-AI-attribution rule

## Roadmap

- `do` / `guide` / `adaptive` operating modes
- Per-repo conventions cache

Considered and rejected:

- **`pipeline` skill.** The `gh search prs --author @me` one-liner above already covers it; a skill wrapper would add ceremony without insight at the contribution volume this plugin is designed for.
- **General contribution-progress journal** (per-PR state file, session log of what happened). Most of what you'd want to persist is already in GitHub (`gh pr view --json reviews,comments,reviewDecision`), the consumer repo (`patches/*.patch` from Phase 7), the `log` skill (portfolio narrative from merged PRs), or the roadmap per-repo conventions cache (accumulated repo knowledge). A parallel side-journal creates a sync problem with GitHub and decays the moment it falls out of date. The one narrow gap — *why you bailed when no PR opened* (Phase 3 tractability gate, Phase 1 step 0a stalled-adjacent-PR) — is small enough to be a one-line append to a `CONTRIBUTIONS.md` in the consumer repo, not a skill.

## Contributing

Issues and PRs welcome. For new skills, open an issue first to align on scope. For fixes, a failing test + minimal diff is the path of least resistance. (The plugin is literally the workflow we want you to use.)

## License

MIT — see [LICENSE](./LICENSE).
