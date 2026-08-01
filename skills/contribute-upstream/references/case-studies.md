# Case studies

Documented failure cases that motivate each gate. Load this file when:

- You need to remember WHY a specific gate exists.
- You're staring at a candidate that looks like one of these shapes and want to verify.
- A user asks "why does the skill check X?"

Each entry is the story; the procedural check stays inline in the SKILL.md phase it belongs to.

---

## Freshness — late assignee (Phase 1 step 0)

**Case:** 2026-05-14, `mastra-ai/mastra#16422`. Chosen from a fresh `find-issues` shortlist; the scout's dup-PR check showed `assignees: []`. ~30 minutes later (after rules-of-the-road scout → clone → reading source → writing the fix → writing tests → commit), the freshness re-check just before push surfaced that `@intojhanurag` had been assigned in the interim. Half a day of work would have shipped as a competing PR.

**Lesson:** Moving the freshness gate to Phase 1 step 0 (before clone or any code-writing) catches this before sunk cost accrues. State changes fast on Hot repos.

---

## Adjacent stalled PR — dead-area signal (Phase 1 step 0a)

**Case:** 2026-05-14, `vercel/ai#13962` passed freshness but the adjacent PR #12924 ("pass abort signal to reconnectToStream so `stop()` works on resumed streams") — same `Chat.stop()` code area — had been open since 2026-02-27 with zero reviews, zero comments, REVIEW_REQUIRED. ~75 days of total maintainer silence in that area.

**Lesson:** A fresh PR for the same area faces the same fate. Zero reviews + zero comments + REVIEW_REQUIRED for 30+ days on a sibling PR is the "dead area" signal. Surface to the user before investing.

---

## Adjacent ACTIVE PR on shared interface — coordination risk (Phase 1 step 0a, inverse shape)

**Case:** 2026-05-19, `topoteretes/cognee#2815` (small feature: plumb `node_name` through `ChunksRetriever`) passed every existing gate — freshness, token dup-PR search, "already fixed on main" — but a Phase 1 manual scan turned up open PR #2712 (`fix: implement include_payload and node_name filter in ChromaDBAdapter`). Active, not stalled: 4 reviews, 6 comments, CodeRabbit auto-paused "because the branch is under active development." It touched 6 files including `cognee/infrastructure/databases/vector/vector_db_interface.py` and the same adapters (`PGVectorAdapter.py`, `LanceDBAdapter.py`) the `ChunksRetriever` fix would call into.

The PRs do not textually overlap — different files, different layer (retriever vs adapter). Token dup search returns nothing on the issue body identifiers (`ChunksRetriever`, `node_name_filter_operator`) because the PR is genuinely about a different surface. But the *interface* the retriever calls is being reshaped mid-review. If #2712 lands first, the retriever needs to match whatever final signature ships. If the retriever PR lands first, #2712 hits conflicts that aren't its fault. Either way, the contributor inherits coordination work that doesn't fit a 1-hour budget.

**Lesson:** The existing Phase 1 step 0a check (`zero reviews + zero comments + REVIEW_REQUIRED for 30+ days`) only catches the *stalled* dead-area shape. An adjacent PR that is **active and touches the interface our fix calls into** is a different drop signal — the risk isn't a dead area, it's a moving target. Extending step 0a:

- After resolving the file(s) your fix will touch, also list the *imports* / interface symbols those files depend on.
- For each, run a quick `gh search prs --repo <owner>/<repo> --state open <symbol>` looking for PRs touching the imported file or interface symbol.
- If a non-stalled PR (recent activity, has reviews/comments) is reshaping the interface you'll call into, surface to the user before clone. The right call is usually one of: (a) wait for that PR to land, (b) comment on your issue asking the maintainer how they want this sequenced, or (c) abort.

**Cheap check:**
```
gh pr view <adjacent-pr> --repo <owner>/<repo> --json files,reviews,comments,updatedAt
# is it active? (reviews/comments non-empty, updatedAt recent)
# does its `files` list overlap interface files your fix imports?
```

---

## Already fixed on main — wrong slice of the version axis (Phase 1 step 0b)

Three same-shape cases on 2026-05-14:

- **`assistant-ui#4009`** — reporter on v0.14.0. dosu bot's follow-up comment said "this has been addressed in recent PRs merged to main: PR #3927 (May 5), PR #3954" — but the issue stayed open because the merged PRs didn't formally close it.
- **`mastra#16383`** — reporter on `@mastra/schema-compat@1.1.3`; current main is 1.2.10 with PR #14624 (2026-04-13) landing "fix(schema-compat): improve provider structured output and tool-call compatibility" 4 weeks before the issue was filed. Existing strict-mode tests in `openai.test.ts` (792/792 green) explicitly assert `.optional()` → strict-mode compliant schema.
- **`drizzle-orm#5755`** — *inverse shape:* reporter on `drizzle-kit@1.0.0-rc.2` (mid-rewrite beta); the named `casing` config feature was wholly absent in the new code paths — NOT already fixed, it was intentionally / accidentally dropped. Same family of "filed against the wrong slice of the version axis," different direction.

**Lesson:** Compare the reporter's version against main. If a fix-shaped commit landed between the reporter's version and main, the issue may already be fixed and the reporter just needs to upgrade. If the feature is wholly absent on main, it's a re-implementation, not a bug fix — switch to Phase 6.

**Cheap check:**
```
gh api 'repos/<owner>/<repo>/commits?path=<file>&per_page=15' \
  --jq '.[] | {sha: .sha[:8], date: .commit.author.date, msg: .commit.message | split("\n")[0]}'
```

---

## Invitation-only upstream — HARD STOP (Phase 1 step 2)

**Case:** `openai/codex` accepts external PRs by invitation only — uninvited PRs are closed unread.

**Lesson:** Scan the contributing doc and PR template for "invitation only" / "do not accept unsolicited" / "closed without review" / "external contributions are closed". If found, STOP — do not proceed to Phase 2 clone. Switch to Phase 6 (issue-only) and offer to comment on the issue with analysis + suggested fix, which is what these projects explicitly invite.

---

## Token-based duplicate-PR search — implementation-titled PRs (Phase 1 step 5)

**Case:** Issue titled "Layouts for paths that start with underscore (%5F)…" had an open PR titled "fix(typegen): normalize %5F to _…" — caught only by the `%5F` literal-token search, not by any title-keyword paraphrase.

**Lesson:** Title-keyword searches miss PRs whose title describes the *implementation* rather than the *symptom*. Search by tokens extracted from the issue body: the issue number itself, URL-encoded or distinctive literals, backticked code identifiers, error message fragments. Title keywords as a last resort, not first.

---

## Tractability gate — feature gone, not bug (Phase 3 step 4)

**Case:** `drizzle-team/drizzle-orm#5755` was framed as a missing type field, but the v1 rewrite removed the entire `casing` runtime pipeline. Adding the type alone would ship a misleading API (TS happy, runtime broken).

**Lesson:** When an issue says "X is missing/broken in version Y" against a `vN.0.0-beta/rc.M` of a package mid-rewrite, briefly check whether X exists in the new code paths. If wholly absent (not just typed-out), the "fix" is a re-implementation, not a bug fix — maintainer intent is required. Switch to Phase 6 (issue-only).

---

## Convention divergence at commit (Phase 3 step 0 + Phase 4 audit)

**Case:** CopilotKit#4798 — first commit had three style divergences from the surrounding file:

1. **Ad-hoc inline cast** instead of the file's pattern of declaring a top-level named type (e.g. `InspectorInternals`) and using a helper function.
2. **Fetch stub repeated per test** instead of `beforeEach` alongside other setup.
3. **Raw Map exposed** from the mock factory instead of controller functions (`emitX`, `setY`) like the existing factories used.

User pushed back, required a cleanup follow-up commit that produced review churn.

**Lesson:** The convention scan (Phase 3 step 0) would have flagged all three before they were written. The convention audit (Phase 4) is verification; the scan is prevention. Both passes use `references/convention-checklist.md`.

---

## Late dup-PR — scout-to-handoff gap (find-issues Phase 4)

**Case:** 2026-05-13 hunt — `mastra-ai/mastra#16514` passed the scout's dup-PR check; PR #16545 ("fix(durable): handle object form of instructions in preparation.ts") opened ~10 minutes later. Caught only when the user explicitly asked "did you check no existing PR?"

**Lesson:** Phase 2/3 scouts may have run minutes ago. On Hot repos, an exact-fix PR can land in that window. Before emitting the ranked list in Phase 4, re-run the dup-PR search against the top-5 candidates in a single batched message. Drop any candidate where a new PR has appeared.

---

## Label-scoped invitation-only convention (Phase 1 step 0c)

**Case:** 2026-05-22, `ChromeDevTools/chrome-devtools-mcp` — two PRs (#2098, #2099) opened against issues carrying the `evals` label (a focused cluster of `evaluate_script` tool-description and CLI-quoting bugs). Both passed every existing Phase 1 gate: contributing policy was open, no invitation-only prose in CONTRIBUTING or the PR template, repo was active with recent merged external PRs, no dup PRs, all CI green, CLA signed. Within hours, the same maintainer (`OrKoN`) closed both with effectively identical wording:

> "the difficulty here is not in generating a fix but running evals to make sure they work. I will close it in favour of the team handling the bugs marked with the 'evals' label."

Two same-day closures with matching language = an enforced policy that is **scoped to the label**, not to the repo or contributor. Nothing in any doc surface said so.

**Lesson:** Invitation-only conventions can be label-scoped. Before clone, sample recent closed-unmerged PRs that linked an issue with the same label as the candidate issue, and look for matching closer-comment shapes. Two closures with the same wording = drop. One closure could be idiosyncratic. The cheap query (`gh search prs --repo <r> --state closed "is:unmerged label:<label>"`) takes one round-trip.

---

## Mid-review competing merge (Phase 8 step 0)

**Case:** 2026-05-13 → 2026-05-22, `better-auth/better-auth#9605`. Opened against issue #9412 ("SSO OIDC callback redirects with `?error=signup disabled` — space-encoded, inconsistent with other auth callbacks"). Fix: normalize the error to underscores (`signup_disabled`) before appending to the redirect URL. Passed every Phase 1 gate at open time.

Sat in review for 9 days with zero human engagement. On 2026-05-22 the maintainer merged a different fix to the same file (`#9722: fix(sso): url-encode error query value in OIDC callback redirect`) — URLSearchParams + URL-encoding (`signup%20disabled`) rather than normalization. Both approaches unblock the symptom; the maintainer chose theirs. `#9605` became `mergeable_state: dirty` and functionally obsolete the moment `#9722` landed.

The freshness gate at Phase 1 step 0 wouldn't have caught this because the competing PR didn't exist at open time. The Phase 8 review-response loop also wouldn't catch it because there was no review to respond to — silence, then obsolescence.

**Lesson:** Before doing anything else in Phase 8 — fetching review state, classifying feedback, or planning a push — list commits to `main` on the same files since the PR was opened. If a competing fix has landed, the right move is to close the PR gracefully with a comment pointing at the merged PR. Continuing to iterate on review comments after the bug is fixed wastes the reviewer's and the user's time.

---

## Dead target — fixing code nothing calls (Phase 1 step 5b + Phase 3 step 4)

**Case:** 2026-05-15 → 2026-08-01, `CopilotKit/CopilotKit#4842`. Opened against issue #4772 ("retry-utils.ts — Retry-After > 60s throws instead of surfacing STOP condition"). The fix added a typed `RetryAfterExceededError` to `packages/runtime/src/lib/runtime/retry-utils.ts` so callers could discriminate on `instanceof` instead of parsing a message string, with tests. Every existing Phase 1 gate passed: open contributing policy, no dup PR, CLA fine, active repo, and the fix matched the issue's stated request exactly.

68 days later the maintainer requested changes and led with the fundamental problem:

> "**The code path has no consumers.** `fetchWithRetry` (and the whole `retry-utils` module) is orphaned — `git grep` across the monorepo finds it referenced *only* by its own test file… Until the retry utility is actually integrated, this is polishing dead code."

Verified after the fact: a repo-wide code search for `fetchWithRetry` returns three hits — the module, its test, and an unrelated Go docs page. The maintainer explicitly called the patch itself "clean and well-tested". The code was never the problem; the *target* was.

Two compounding misses in the same PR:

- **A `.changeset/` file was added to a repo that no longer uses changesets.** `.changeset/` is 404 on `main` (they moved to conventional-commit releases), so the PR carried a file that could not merge. Phase 1 reads `.changeset/config.json` — a 404 there is a signal to *stop* adding changesets, not a lookup that silently failed.
- **The issue's cited path did not exist.** #4772 pointed at `packages/runtime/src/util/retry-utils.ts`; that path 404s. The real file lives under `src/lib/runtime/`. The issue was filed by an author with exactly one issue each across a dozen unrelated repos (`helicone`, `crawlee`, `cline`, `dust`, `voltagent`, …), with body boilerplate reading "Corpus reference:" and "Related pattern:" — a bulk automated scan, not a user who hit the bug.

**Lesson:** An issue describing a real defect in a real file is not evidence that fixing it changes anything. Before writing the fix, prove the symbol is reachable from something a user runs: `git grep -n '<symbol>' -- . ':!*test*' ':!*spec*'`. Definition + own tests only = orphaned; switch to Phase 6 and ask whether the module is meant to be wired in. And when the issue came from someone who didn't hit the bug themselves, treat every premise in it — file path included — as unverified until you check it. Cost of the missing check: one `git grep` versus 68 days of a reviewer's queue and a closed PR.
