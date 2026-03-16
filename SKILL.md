---
name: plan-reviewer
description: "Review and strengthen an implementation plan. Use this skill whenever the user says 'review the plan', 'check the plan', 'review my plan', 'find gaps in the plan', or asks you to audit, validate, or improve an existing implementation plan. This skill performs a structured review: gap analysis, edge case discovery, UI challenge identification, dry-run simulation, plan updates, and question generation, by cross-referencing the plan against the actual project codebase. Supports three review depths: quick, standard (default), and deep."
---

# Plan Reviewer

You are reviewing an implementation plan. Your job is to pressure-test this plan against the real codebase, surface what's missing or risky, and produce an improved version the team can confidently execute on.

## How to Find the Plan

1. If the user points you to a specific file, use that.
2. Look for `implementation_plan.md` in the current working directory, project root, or any agent-specific artifact directory.
3. Search the project for files matching `*implementation_plan*` or `*impl_plan*`.
4. If you still can't find it, ask the user where the plan is located.
5. Read the **entire** plan before starting the review.

## Review Depth

This skill supports three review depths. The user can request a specific depth, or you can infer it from context. **Default is `standard`.**

| Depth | Phases Run | When to Use | Context Cost |
|---|---|---|---|
| `quick` | 1 → 2 → 5 → 6 | Small changes, config updates, low-risk work | Low |
| `standard` | 1 → 2 → 3 → 5 → 6 | Most implementation plans (default) | Medium |
| `deep` | 1 → 2 → 3 → 4 → 5 → 6 | Large features, risky refactors, anything touching UI | High |

**How to determine depth**:
- If the user says "quick review", "fast review", or "just check for gaps" → use `quick`
- If the user says "deep review", "thorough review", or "full review" → use `deep`
- If the plan involves UI/frontend changes → recommend `deep` (Phase 4 is only included in deep)
- Otherwise → use `standard`

Phase 7 (Questions) always runs at the end regardless of depth.

---

## Review Phases

Work through each phase that applies to your selected depth, in order. Don't skip an included phase even if you think it doesn't apply — document "No issues found" if a phase turns up clean.

---

### Phase 1: Understand the Plan
**Included in**: `quick` · `standard` · `deep`

Read the implementation plan end-to-end. Before reviewing, make sure you can answer:

- What problem is being solved?
- What files/modules are being changed?
- What's the expected behavior after the change?
- What does the verification plan look like?

Summarize your understanding in 2-3 sentences. This grounds everything that follows.

---

### Phase 2: Gap Analysis
**Included in**: `quick` · `standard` · `deep`

Cross-reference the plan against the **actual project code**. For every file the plan mentions modifying, read that file. Then look for gaps:

- **Missing file changes**: Does the plan touch file A but forget that file B imports from A and also needs updating? Trace direct callers and importers (one level deep for `quick`, full chain for `standard`/`deep`).
- **Missing error handling**: Does the plan add a new code path but not describe what happens when it fails?
- **Missing data flow**: Does the plan change a data structure but miss downstream consumers of that structure (other services, database queries, API responses)?
- **Missing config/env changes**: Does the plan require new environment variables, feature flags, or configuration entries that aren't mentioned?
- **Missing migration steps**: Does the plan change database schemas, API contracts, or file formats without describing how existing data transitions?
- **Incomplete verification**: Does the verification plan actually test the scenarios described in the changes? Are there untested paths?

**How to investigate**: Search the codebase for usages of changed functions, types, and modules. Trace import chains and callers — don't just trust the plan's file list, look for what it missed.

---

### Phase 3: Edge Case Discovery
**Included in**: `standard` · `deep`
**Skipped in**: `quick`

For each change in the plan, think about what could go wrong at the boundaries:

- **Concurrency**: Can two processes hit this code path simultaneously? What happens?
- **Empty/null/undefined inputs**: What if a value the plan assumes exists is actually missing?
- **Boundary values**: What happens at zero, negative numbers, max values, empty arrays, empty strings?
- **Partial failures**: If step 2 of a 3-step operation fails, what state is the system left in? Can it recover?
- **Race conditions**: Are there async operations that could resolve in an unexpected order?
- **Retry behavior**: If this operation is retried, does it produce the same result (idempotency)?
- **Backward compatibility**: Can old clients/data still work after this change?
- **Timeout and network failures**: What happens if an external call hangs or times out?

Don't invent impossible scenarios — ground your edge cases in what the code actually does. Read the real implementations to understand actual failure modes.

---

### Phase 4: UI Challenge Identification
**Included in**: `deep`
**Skipped in**: `quick` · `standard`

If the plan involves any frontend, UI, or user-facing changes, evaluate:

- **Loading states**: What does the user see while data is being fetched? Is there a skeleton/spinner/placeholder?
- **Error states**: What does the user see when something fails? Is the error message actionable?
- **Empty states**: What does the UI look like when there's no data yet?
- **Responsiveness**: Will this work on mobile? On different screen sizes?
- **Accessibility**: Are interactive elements keyboard-navigable? Do they have proper ARIA labels?
- **Optimistic updates vs. server roundtrips**: Should the UI update immediately or wait for confirmation?
- **State consistency**: If the user has multiple tabs open, or navigates away and back, does the UI stay consistent?
- **Animation/transition jank**: Will layout shifts or re-renders cause visual glitches?

If the plan has no UI component, document "No UI changes — phase skipped" and move on.

---

### Phase 5: Dry-Run Simulation
**Included in**: `quick` · `standard` · `deep`

Mentally execute the plan step-by-step against the real codebase — as if you were implementing it right now, but without writing any code. The depth of simulation varies:

- **`quick`**: Read each modified function and verify the plan's described changes are coherent with the current code. Check for obvious mismatches (wrong variable names, missing imports, type errors).
- **`standard`**: Do everything in `quick`, plus trace one level of callers to verify they handle the changes correctly. Simulate one failure scenario for the most critical change.
- **`deep`**: Full simulation. For each proposed change, follow the complete checklist below.

#### Full Simulation Checklist (deep only)

For each proposed change:

1. **Open the real file** and read the function/block being modified.
2. **Trace the execution path**: Walk through the code line-by-line. What gets called? What gets returned? What state changes?
3. **Follow the data**: Track variables from input to output. Where does a value come from? Where does it go next? Does the plan account for every hop?
4. **Simulate the caller**: Who calls this function? Read the caller and check — does it handle the new return value or error? Does it pass the right arguments after the change?
5. **Simulate failure mid-flow**: Pick the most critical point in the change and imagine it throws. Trace what happens upstream. Does the caller catch it? Does the database get left in a dirty state? Does a queue message get acked but never processed?
6. **Check ordering assumptions**: Does the plan assume step A completes before step B? Is that actually guaranteed in the code, or are they async/parallel?

**What to look for** (all depths):
- Variables the plan references that don't exist in the actual function signature
- Type mismatches between what's returned and what the caller expects
- Missing `await` on async calls the plan adds
- Database writes that happen before validation, leaving dirty data on failure
- Event/queue messages sent before the transaction commits
- Import statements that will be needed but aren't mentioned in the plan

Document each finding with the exact file, line, and the specific execution path that breaks.

---

### Phase 6: Update the Plan
**Included in**: `quick` · `standard` · `deep`

Now take everything you found in previous phases and **update the implementation plan**. Don't just list problems — fix them:

1. Add missing file changes to the proposed changes section
2. Add edge case handling to the relevant file change descriptions
3. Add UI state handling (loading, error, empty) where needed
4. Add fixes for issues found during dry-run simulation
5. Update the verification plan to cover the new scenarios
6. Add any missing migration or config steps

Match the structure and formatting conventions already used in the plan. Mark your additions clearly so the user can see what changed:

```markdown
> [!IMPORTANT]
> **Added by review**: [description of what was added and why]
```

Write the updated plan back to the same file.

---

### Phase 7: Questions for the User
**Always runs** regardless of depth.

After updating the plan, list any remaining questions — things you genuinely couldn't resolve from the code alone:

- Ambiguous business requirements ("Should inactive users also get this notification?")
- Design tradeoffs where both options are reasonable ("Should we retry 3 times or fail fast?")
- Missing context ("I can't determine the expected behavior when X happens — what should it be?")

**Rules for questions**:
- Only ask questions you actually need answered. Don't pad with obvious ones.
- Frame each question with enough context that the user can answer without re-reading the whole plan.
- If there are no genuine questions, say so — don't invent questions to fill the section.

Format as a numbered list at the end of the updated plan under a `## Review Questions` section.

---

## Output Format

After completing all phases, produce a **review summary** at the top of the updated plan file:

```markdown
## Plan Review Summary

**Reviewed**: [date]
**Review depth**: [quick | standard | deep]
**Gaps found**: [count]
**Edge cases identified**: [count or "skipped"]  
**Simulation issues**: [count]
**UI challenges**: [count or "skipped"]
**Questions for user**: [count or "None"]

### Key Findings
- [One-liner for each significant finding, grouped by phase]
```

Then the rest of the updated plan follows below, with additions clearly marked.

---

## Important Reminders

- **Read the actual code, not just the plan.** The plan might describe something that doesn't match reality. Trust the code.
- **Be specific.** Don't say "error handling might be missing." Say "the `executeSwap` function in `SwapService.ts:142` catches `Error` but doesn't handle the `InsufficientFundsError` subclass that `checkBalance` can throw."
- **Prioritize.** If you find 20 issues, call out the top 3-5 that could actually cause incidents. Don't bury critical issues in a sea of nits.
- **Stay grounded.** Only flag issues you can demonstrate from the code. Hypothetical concerns that have no basis in the actual implementation waste everyone's time.
- **Respect the depth.** Don't over-investigate on a `quick` review or under-investigate on a `deep` one. Match your effort to the selected depth.
