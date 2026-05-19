# oss-contribute profile

This is an example of the shared profile that `find-issues`, `contribute-upstream`, and `profile` all read from. Copy it to your active profile location (see below) and edit.

**Profile location** (resolved in this order, first existing wins):

1. `$CLAUDE_PLUGIN_DATA/profile.md` — when installed as a plugin
2. `~/.claude/plugins/data/oss-contribute/profile.md`
3. `~/.claude/skills/oss-contribute/profile.md` — local-development mode

To create or edit the active profile, run `/oss-contribute:profile edit`.

---

# oss-contribute profile

## Watched repos
- better-auth/better-auth
- vercel/next.js
- vitest-dev/vitest
- vercel/ai
- prisma/prisma

## Languages
typescript, go, python

## Stack
next.js, react, prisma, vitest, drizzle, tailwind

## Default GitHub account
shiminshen

## Git commit identity
Used to set repo-local `git config user.name` / `user.email` after each `git clone`, so OSS commits aren't stamped with your global (often work) identity. Prefer the GitHub noreply form to keep your real email private.
- name: shiminshen
- email: 16914659+shiminshen@users.noreply.github.com

(Replace `16914659` with your numeric GitHub user ID — find it with `gh api user --jq .id` while logged in as the personal account.)

## Default budget
1h

## What "ripe" means to me
Small diff (<50 lines). Clear repro in the issue body. No assignee. No linked PR.
Maintainer has triaged (has a label). Repo has merged something in the last 30 days.
