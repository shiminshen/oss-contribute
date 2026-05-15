# Contributing to oss-contribute

Thanks for considering a contribution. This plugin's whole point is to lower the friction of upstream contributions — so the bar for contributing back to it should be low too. Read this once, then [the plugin is literally the workflow we want you to use](./README.md#the-reactive-flow-killer-feature) for the actual change.

## TL;DR

- **Bugs**: file an issue with reproduction steps. A failing case beats a paragraph.
- **New skill or behavioural change**: open an issue first to align on scope. Don't write the markdown until we agree it belongs.
- **Small fixes** (typo, wording, broken link, clarification): straight to PR, no issue needed.
- **Editing an existing skill**: read [`CLAUDE.md`](./CLAUDE.md) first — it documents the conventions the skills enforce on users, which apply to edits here too.

## What this repo is

Pure markdown. No build system, no runtime, no test suite. Everything ships as `SKILL.md` files loaded by Claude Code at invoke time.

There is no `package.json`, no `node_modules`, no linter. Don't add one without explicit reason — the absence is deliberate.

## Repository layout

- `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` — manifests. **Both have a `version` field that must stay in sync** on every release.
- `skills/<name>/SKILL.md` — one skill per directory. YAML frontmatter (`name`, `description`, `trigger`) is what Claude Code reads to register the skill; the body is what gets injected when invoked.
- `skills/contribute-upstream/references/*.md` — auxiliary content loaded on-demand, not on-invoke. See [Token budget](#token-budget) below.
- `docs/profile.example.md` — example of the shared profile all three skills read from.
- `CHANGELOG.md` — Keep-a-Changelog format, semver.

## Editing workflow

1. **Edit** the relevant `SKILL.md` or `references/*.md`. Match the existing voice: terse, second-person imperative. "Do X" not "You should consider doing X".
2. **If you change behaviour**:
   - Bump `version` in **both** `.claude-plugin/plugin.json` AND `.claude-plugin/marketplace.json`. They must match.
   - Add a `CHANGELOG.md` entry under a new version header (Keep-a-Changelog, semver). Look at recent entries for tone — they explain the motivating case and tradeoff, not just the diff.
3. **Verify** by re-reading the changed phase end-to-end. When possible, try the skill via `/oss-contribute:<skill>` in a Claude Code session with the plugin installed locally.

There is no test suite to run. The verification is human re-read + a real invocation.

## Token budget

`contribute-upstream` is the large one (~7.2k on-invoke as of v0.6.0). Comparable plugins top out around 3.9k, so we care about this number.

When adding content, decide where it belongs:

- **Inline in `SKILL.md`** — procedural gates the model needs to know *what to check*: BLOCKING criteria, drop conditions, cheap-check commands.
- **In `references/*.md`** — narrative "why this gate exists" content: failure-case stories, full checklists, the full Phase 8 procedure. Loaded only when the model recognises a similar shape or enters that phase.

One-line motivating-case pointers in the inline gates point at `references/case-studies.md#<anchor>`. Keep that pattern.

If your addition makes the on-invoke cost grow noticeably, justify it in the PR description.

## Hard rules (do not weaken)

These are rules the plugin enforces on behalf of users. Don't relax them without an explicit reason in the PR and a CHANGELOG entry:

- **No AI-attribution trailers on any commit.** No `Co-Authored-By: Claude`, no `Generated-By`, no "AI assisted" markers — anywhere, including commit bodies and PR bodies. This applies to commits in *this* repo too. It would be embarrassing for the plugin that enforces sole-authored OSS contributions to violate the rule itself.
- **Never push to an upstream's main repo.** Always work from the user's fork.
- **Pre-PR confirmation gate is BLOCKING.** `gh pr create` never runs without the user seeing the full body + commits + files and explicitly saying yes.
- **Invitation-only repos are a HARD STOP.** Detection happens in Phase 1; do not proceed to Phase 2.
- **Surface, don't suppress.** If a contribution probably won't be accepted (dormant repo, scope mismatch, CLA blocker, duplicate PR, re-implementation disguised as bug fix), the skill says so before doing the work.

## Procedures with a canonical home

Some procedures appear across multiple skills. Each has **one authoritative version**; the other skills defer via a one-line pointer. When editing such a procedure, find the canonical version first — don't fork.

The canonical home is usually inside `contribute-upstream` (since it's the heavy skill). If you're not sure, grep for the procedure name across skills and follow the pointers.

## Commit & PR style

Conventional Commits:

```
<type>(<scope>): <subject>

<optional body>
```

Types: `feat`, `fix`, `refactor`, `docs`, `chore`. Scope is usually the skill name (`contribute-upstream`, `find-issues`, `profile`) or `plugin` for cross-cutting changes.

Subject line describes the user-visible change, not the internal mechanism. Look at recent `git log` for tone.

**No AI co-author trailers.** (See [Hard rules](#hard-rules-do-not-weaken).)

PR description should answer:

1. What user-visible behaviour changes?
2. What motivated it (real case, not hypothetical)?
3. What's the on-invoke token impact, if any?

## PR checklist

Copy this into your PR description and tick before requesting review:

- [ ] Behaviour change: bumped `version` in `.claude-plugin/plugin.json` AND `.claude-plugin/marketplace.json` (matching)
- [ ] Behaviour change: added a `CHANGELOG.md` entry under the new version header
- [ ] New content placed in the right tier (inline for procedural, `references/` for narrative)
- [ ] Voice matches existing skills (terse, second-person imperative)
- [ ] No AI-attribution trailers in commits or PR body
- [ ] Tried the skill locally via `/oss-contribute:<skill>` if behaviour changed

## Reporting bugs

File an issue with:

1. What you ran (the exact `/oss-contribute:...` invocation)
2. What you expected
3. What happened
4. Your environment: OS, Claude Code version, which `gh` accounts are logged in (don't paste tokens)

If the bug is "the skill suggested the wrong thing", paste the model's reasoning if you can — it tells us whether the issue is in the skill content or in model behaviour we can't directly control.

## Proposing a new skill

The bar is "what user problem does this solve that the existing three skills don't, and why is it skill-shaped rather than a flag on an existing skill?".

Look at the [Roadmap](./README.md#roadmap) and [Considered and rejected](./README.md#roadmap) section first — we explicitly document things we decided *not* to build and why. If your idea is in the rejected list, address why the rejection reasoning doesn't apply to your version.

Open an issue first. Don't write the SKILL.md until we agree on scope.

## Questions

Open a GitHub Discussion (or an issue tagged `question`). The "is this a good fit for the plugin?" conversation is much cheaper than a closed PR.
