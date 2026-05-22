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
