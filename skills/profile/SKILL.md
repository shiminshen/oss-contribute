---
name: profile
description: View or edit the shared profile that the oss-contribute plugin uses for stable preferences (watched repos, languages, default GitHub account, default budget, what "ripe" means). All other skills in the plugin read from this same file so settings never drift across surfaces.
trigger: /oss-contribute:profile
---

# profile

Manages the shared `oss-contribute` profile. All other skills in the plugin (`find-issues`, `contribute-upstream`) read from this same file.

## Usage

```
/oss-contribute:profile               # show current profile (default)
/oss-contribute:profile show          # same as above
/oss-contribute:profile edit          # open in $EDITOR (or run interactive setup if missing)
```

## Profile location

Resolve in this order and use the **first** one that exists. If none exist and the subcommand is `show`, report which paths were checked; if `edit`, create the first one (plugin data dir).

1. `$CLAUDE_PLUGIN_DATA/profile.md` — when running as an installed plugin (Claude Code sets this env var). Survives plugin updates.
2. `~/.claude/plugins/data/oss-contribute/profile.md` — same path as #1, for explicitness.
3. `~/.claude/skills/oss-contribute/profile.md` — local-development fallback.

## Phase A — `show` (or no arg)

Print the current profile and the file path it was loaded from. Read-only.

If no profile exists, print:

```
No profile found. Checked:
  - $CLAUDE_PLUGIN_DATA/profile.md
  - ~/.claude/plugins/data/oss-contribute/profile.md
  - ~/.claude/skills/oss-contribute/profile.md

Run: /oss-contribute:profile edit  to set one up.
```

## Phase B — `edit`

If the profile exists, open it in `$EDITOR` (or `vi` if `$EDITOR` is unset).

If the profile is missing, walk through interactive setup, asking one question at a time:

1. Which GitHub repos do you want to watch? (`owner/repo` list)
2. Which languages / frameworks do you work in?
3. Which `gh`-logged-in account should the plugin fork/PR from? (run `gh auth status` and let the user pick)
4. Default time budget? (`30m` / `1h` / `half-day` / `weekend`)
5. Anything specific that makes an issue feel "ripe" for you? (free text)

Write to `$CLAUDE_PLUGIN_DATA/profile.md` (or the local-mode path if that env var is unset) using the schema below. Confirm contents with the user before writing.

## Profile schema

```markdown
# oss-contribute profile

## Watched repos
- owner/repo
- ...

## Languages
typescript, go, python

## Stack
next.js, react, prisma, vitest

## Default GitHub account
shiminshen

## Default budget
1h

## Follow-up
(optional — `follow-up` only; omit for defaults)
- stale window: 14d
- ignore comment authors: vercel, coderabbitai

## What "ripe" means to me
Small diff (<50 lines). Clear repro in the issue body. No assignee. No linked PR.
Maintainer has triaged (has a label). Repo has merged something in the last 30 days.
```

## Hard rules

- **`show` is read-only.** Never modify the profile in show mode.
- **Profile is the source of truth.** Other skills must not silently override profile values from args; conflicts surface to the user.
- **No remote secrets.** The profile is a local preferences file. Do not store API tokens or credentials here — `gh` handles GitHub auth, and that's where account selection terminates.
