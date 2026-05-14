# Convention checklist

Used by **Phase 3 step 0** (pre-coding scan — prevention) and **Phase 4 Convention audit** (post-write verification). Same checklist, two passes.

Your new code must match the patterns the maintainers already chose — not just compile and pass tests. The scan absorbs the patterns before you write; the audit verifies your output matches.

## The seven dimensions

### Test file structure

Open the closest existing test file end-to-end. Note:

- `describe` / `it` shape (nested? flat? grouped by symbol or by behavior?)
- Setup placement: `beforeEach` (top-level / nested) vs inline per test
- Assertion style: `expect(...)` chains, custom matchers, `assert(...)`, etc.
- Arrange / Act / Assert organization — blank-line separated? Comment-marked? Inline?

### Helpers and mock factories

- Do existing helpers return raw state (Maps, arrays) or only controller functions (`emitX`, `setY`)?
- Top-level factory functions vs ad-hoc inline mocks?
- Reuse existing helpers; **don't invent a parallel one** with a slightly different shape.

### Naming

- Verb forms used by the file: `getX` / `createX` / `emitX` / `buildX`? Pick the dominant form.
- Top-level named types declared for casts (e.g. `InspectorInternals`) vs inline `as` casts? Match the dominant pattern.

### Cross-cutting setup

Global stubs (`fetch`, timers, env, randomness, network) — are they in a top-level `beforeEach` alongside other setup, or inline per test? Match it.

### Comment density and tone

Does the file explain WHY each test exists with a multi-line preamble, or one-liners, or no comments? Match it.

### Lifecycle / async patterns

Is there a standard "after every state change, `await component.updateComplete`" (or equivalent for the framework) idiom? Apply it without being asked.

### Import style

- Relative vs alias paths (`@/foo` vs `../foo`)?
- Type-only imports (`import type {}`) separated or merged?
- Group ordering (external → internal → relative → type)?

Match the file you're touching.

## Sourcing the conventions

Conventions in older files may differ from what the maintainers *enforce in review*. When in doubt, run:

```
gh pr list --search "<file/path>" --state merged --limit 3
```

Read the diffs and review threads — those show what conventions are actually enforced *today*, not what existed when older files were written.

## Two-pass usage

| Pass | When | What to do |
|---|---|---|
| **Phase 3 step 0** | Before writing the failing test or any fix code | Read the file end-to-end + recent merged PRs. Capture patterns. |
| **Phase 4 audit** | After writing the fix, before commit | Re-read your diff against the same checklist. Catch what slipped through. |

The audit confirms the scan. Skipping the scan means the audit catches mistakes after they're written, producing review churn.

Motivating case: **CopilotKit#4798** — see `references/case-studies.md`.
