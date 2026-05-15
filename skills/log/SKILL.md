---
name: log
description: Generate a portfolio entry from your merged upstream contributions. Reads the active GitHub account from the shared profile, queries `gh search prs --author @me --state merged` for a configurable window (default 90 days), and renders a hybrid artifact — table-of-contents at the top, one detail block per PR below. Sources content from PR title + body verbatim; no embellishment, no auto-publishing.
trigger: /oss-contribute:log
---

# log

Turn the merged-PR history of your GitHub account into a single portfolio artifact you can paste into a CV section, a quarterly recap, or your own notes.

This is a **read-only** skill. It does not post to LinkedIn, GitHub, or anywhere external. It writes a local file (or stdout) that you review and decide what to do with.

## Usage

```
/oss-contribute:log                                   # last 90 days, default account
/oss-contribute:log --since 2026-01-01                # explicit start date
/oss-contribute:log --since 30d                       # relative window (Nd, Nw, Nm, Ny)
/oss-contribute:log --account <gh-login>              # override profile account
/oss-contribute:log --stdout                          # print instead of asking
```

## Profile location

Read the profile in this order:

1. `$CLAUDE_PLUGIN_DATA/profile.md` — when running as an installed plugin
2. `~/.claude/plugins/data/oss-contribute/profile.md` — fallback for direct-installed plugins
3. `~/.claude/skills/oss-contribute/profile.md` — fallback for local-development mode

If none exist, dispatch to the `profile` skill to set one up before continuing — the skill needs at least the **Default GitHub account** field.

## Phase 1 — Resolve the account

1. Load the profile. Read **Default GitHub account**.
2. If `--account` was passed, use it instead (no profile mutation — per-invocation override only).
3. If multiple `gh` accounts are logged in and the profile has no default, run `gh auth status` and ask the user which one to use. **Ask every time** when ambiguous — do not silently fall back to the active account.
4. If the resolved account is not currently active, switch to it for the duration of this skill: `gh auth switch -u <login>`. Restore the original active account at the end (even on failure — use a trap or always-runs cleanup).

## Phase 2 — Fetch merged PRs

Resolve the window:

- `--since YYYY-MM-DD` → use verbatim.
- `--since 30d` / `7w` / `3m` / `1y` → compute the date from today.
- No flag → default to 90 days back from today.

One `gh` call:

```bash
gh search prs --author "@me" --state merged \
  --merged ">=$WINDOW_START" \
  --sort updated --order desc --limit 100 \
  --json url,title,repository,number,mergedAt,additions,deletions,changedFiles,body
```

If `--limit 100` is hit, warn the user and suggest a tighter window — the skill does not auto-paginate, because a portfolio entry that doesn't fit in one window is not a portfolio entry.

## Phase 3 — Render the artifact

Hybrid format: table-of-contents at the top, one detail block per PR below.

```markdown
# Contributions — <window label>

| Date | Repo | PR | Diff |
|---|---|---|---|
| <merged-date> | <owner/repo> | <#N> | +<add>/-<del> |
| ... |

---

## <owner/repo>#<N>

**Merged** <merged-date> · **Diff** +<add>/-<del> across <N> files

<PR title>

<first non-empty paragraph of PR body, verbatim — see "Body extraction" below>

Link: <PR url>

---

## ...
```

### Window label

- `--since YYYY-MM-DD` → `"Since <date>"`
- `--since 90d` (default) → `"Last 90 days"` or quarter label if it lines up (`"2026-Q2"` when the window approximates one calendar quarter)
- `--since 1y` → `"Last 12 months"`

The label is cosmetic — when in doubt, just use `"<start-date> → <today>"`.

### Body extraction

Source from PR `title` + `body` verbatim. Do **not** invent context, do **not** infer what the bug was beyond what the body literally says.

For the body block:

1. Strip GitHub PR-template scaffolding: `## Test plan`, `## Checklist`, `<!-- ... -->` HTML comments, image markdown that points at user-attachments, AI-attribution trailers (the plugin's own rule applies in reverse here — if the body has one, drop the line).
2. Take the first non-empty paragraph that remains. Cap at ~400 chars; truncate with `…` if longer.
3. If the body is empty after stripping, fall back to the title alone — no fabricated "what the fix shipped" narrative.

### Sort order

Detail blocks: newest merged first (matches the table). Easy mental model — the table and the blocks are in the same order.

## Phase 4 — Output

Ask the user once:

> Write to `~/Documents/oss-contribute/log-<YYYY-MM-DD>.md` or print to stdout?

Default to file if the user just hits enter — they can always print after. If `--stdout` was passed, skip the prompt and print.

When writing to a file:

1. `mkdir -p ~/Documents/oss-contribute/` (idempotent).
2. Filename: `log-<today-YYYY-MM-DD>.md`. If a file with that name already exists, append a numeric suffix (`-2`, `-3`, ...) rather than overwriting.
3. Print the absolute path of the file you wrote.

Do **not** open the file, post it anywhere, or `git add` it. This is a local artifact.

## Hard rules

- **No invented data.** Title, body, dates, diff size — all from `gh` verbatim. No "this fix improved performance by X%" unless the PR body says so literally.
- **No auto-publishing.** Stdout or local file only. The skill never posts to LinkedIn, X, a GitHub Gist, the consumer repo, or anywhere external.
- **Ask every time when accounts are ambiguous.** If profile has no default and multiple `gh` accounts are logged in, prompt — don't fall back.
- **Restore the active gh account.** If the skill switched accounts in Phase 1, it switches back at the end. Even on failure.
- **No PR-body fabrication.** If the body is empty or templated-only, fall back to the title. Do not write "this PR fixes a bug in X by doing Y" inferred from the diff stats.
- **Read-only.** The skill never modifies the profile, never opens a PR, never writes to the consumer repo. Only writes the one log file (or stdout).

## When this skill is the wrong tool

- **You want a journal of every contribution attempt, including bails.** Out of scope — the rejected-`pipeline` reasoning in the README applies. Append a one-liner to `CONTRIBUTIONS.md` in the consumer repo instead.
- **You want to *find* something to contribute to.** Use `/oss-contribute:find-issues`.
- **You want to *ship* an upstream fix.** Use `/oss-contribute:contribute-upstream`.
- **You want a status of *open* PRs (not merged).** Out of scope — `gh search prs --author @me --state open` is one line, no skill needed (see README "Checking your pipeline").
