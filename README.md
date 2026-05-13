# oss-contribute

A Claude Code plugin for the **OSS-contribution-as-career-ops** workflow.

Turn the bugs you hit in third-party dependencies into upstream contribution PRs — without leaving your project. The plugin handles the boring parts (reading `CONTRIBUTING.md`, dedup search, package-manager setup, commit and PR conventions, changesets) so you can stay focused on the actual fix.

## What's inside

| Skill | Trigger | Job |
|---|---|---|
| `find-issues` | `/oss-contribute:find-issues` | Ranked shortlist of ripe issues across your watched repos. Read-only. |
| `contribute-upstream` | `/oss-contribute:contribute-upstream <pkg>` | Pre-flight → repro → fix → PR (with confirmation gate). |
| `profile` | `/oss-contribute:profile` | View/edit the shared preferences file. |

All three skills read from one shared profile so settings never drift.

## Install

### From this repo as your own marketplace

```bash
# In Claude Code:
/plugin marketplace add shiminshen/oss-contribute
/plugin install oss-contribute@oss-contribute
```

### From a local clone (for development)

```bash
git clone https://github.com/shiminshen/oss-contribute.git
cd oss-contribute

# In Claude Code:
/plugin marketplace add ./
/plugin install oss-contribute@oss-contribute
```

After install, run `/oss-contribute:profile edit` to set up your watched repos and defaults.

## Workflow

There are two entry points depending on whether you have a specific bug in mind:

### Reactive — you hit a bug

You're working in your project, you hit a bug in a third-party package. Invoke from inside that consumer repo:

```
/oss-contribute:contribute-upstream <package-name>
```

The skill walks through a pre-flight gate (read `CONTRIBUTING.md`, dedup search, CLA check, branch target), reproduces the bug in the upstream's own test framework, ships the smallest possible fix, drafts a PR matching the project's conventions (including changeset if used), and waits for your explicit confirmation before opening the PR. If the bug can't be cleanly reproduced upstream, it falls back to filing a well-written issue instead.

After the PR is open, the skill offers to patch your local copy (`pnpm patch` or `pnpm.overrides` → your fork branch) so your project isn't blocked while the PR sits in review.

### Proactive — shopping for something to work on

```
/oss-contribute:find-issues
/oss-contribute:find-issues --lang typescript --budget 1h
```

Searches your watched repos for issues with `good first issue` / `help wanted` / `bug` labels, drops anything already claimed (assignee, linked PR, "we're working on this" comments), and ranks the rest by repro quality, scope, stack match, and repo health. Hands off to `contribute-upstream` once you pick one — no auto-invoke.

## Profile

The shared profile lives at:

- `$CLAUDE_PLUGIN_DATA/profile.md` when installed as a plugin
- `~/.claude/skills/oss-contribute/profile.md` in local-development mode

See [`docs/profile.example.md`](./docs/profile.example.md) for the schema. Edit via `/oss-contribute:profile edit`.

## Hard rules baked in

- **Never** push to upstream main. Always work from your fork.
- **Never** open a PR without showing the full body and getting your explicit confirmation.
- **Never** assume which GitHub account to use. If multiple are logged in, asks every time.
- **Surface, don't suppress.** If a contribution probably won't be accepted (dormant repo, scope mismatch, CLA blocker, duplicate PR), the skill says so before doing the work.
- Honors each project's AI-contribution policy. If `CONTRIBUTING.md` requires an issue before a feature PR, the skill files or finds one first.

## Receipts

Real contributions shipped using this workflow.

<!-- Add merged PRs here as you ship them. Format: -->
<!-- - [`<owner>/<repo>#<n>`](https://github.com/.../pull/N) — short description (YYYY-MM-DD) -->

<!-- Example: -->
<!-- - [`better-auth/better-auth#9605`](https://github.com/better-auth/better-auth/pull/9605) — normalize OIDC `signup_disabled` error in callback redirect (2026-05-13) -->

## Contributing

Issues and PRs welcome. Before submitting:

- Read [`CONTRIBUTING.md`](./CONTRIBUTING.md) (TODO)
- For new skill ideas, open an issue first so we can align on scope.
- For fixes, a failing test + minimal diff is the path of least resistance.

## License

MIT — see [LICENSE](./LICENSE).
