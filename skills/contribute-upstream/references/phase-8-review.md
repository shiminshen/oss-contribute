# Phase 8 — Respond to PR review

After the PR is open, maintainers will (eventually) review and almost always request changes. This phase handles the round-trip until merge or close.

**Trigger** this phase when `gh pr view <n> --json reviews,comments,reviewRequests` shows new activity since the last check, or when the user invokes `/oss-contribute:contribute-upstream` with the PR URL/number after it's open.

## Steps

1. **Fetch the review state.**

   ```
   gh pr view <n> --repo <upstream> --json \
     reviews,comments,reviewDecision,latestReviews,statusCheckRollup
   ```

   Identify which reviewer requested what, and the disposition (`CHANGES_REQUESTED` / `COMMENTED` / `APPROVED`).

2. **Classify each feedback item.** Bucket every comment / review thread into one of:

   | Bucket | When | Action |
   |---|---|---|
   | **Apply as-is** | Clear, scoped, no design implications | Implement |
   | **Push back politely** | Reviewer is wrong or misread | Reply with counter-argument; do not change code |
   | **Clarify before changing** | Ask is ambiguous | Reply asking a specific question; do not change code yet |
   | **Out of scope for this PR** | Real but unrelated | File as a follow-up issue, link from the comment, do not bloat the PR |

3. **Summarise the bucketed plan to the user** before touching code. ≤10 lines. Get explicit go-ahead.

4. **Apply approved changes.** Smallest possible diff per feedback item. Run the project's tests + typecheck after each round.

5. **Push the update.** `git push origin <branch>` to the **fork** — never to upstream. The PR auto-updates.

6. **Reply to each feedback thread.** For applied items: short ack (`Done in <sha>.`) with a permalink. For pushed-back items: the counter-argument. For clarifying questions: the question.

7. **Re-request review** if the project's convention is to do so explicitly (`gh pr ready` or a comment ping). Match what recent merged PRs in the repo do — don't invent a convention.

## Hard rules for Phase 8

- **No silent re-pushing.** Every push must be paired with a reply on the review thread explaining what changed.
- **Don't rebase the PR branch onto upstream main mid-review** unless the maintainer asks. Adds churn and may invalidate prior reviews.
- **Don't force-push** unless the maintainer asks. Use additive commits; let them squash on merge.
- **Stop and escalate** if the reviewer asks for something that would break Phase 1 hard rules (e.g. AI-attribution trailer, force-push to main, bypass CLA). Surface the conflict to the user; do not silently comply.

## Output (when Phase 8 ran)

- Open feedback threads remaining
- Last push SHA on the PR branch
- Current `reviewDecision`
