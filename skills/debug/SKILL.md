---
name: debug
description: >
  A structured debugging skill for methodically diagnosing and fixing bugs. Trigger this skill
  whenever the user reports a bug, unexpected behavior, error, crash, wrong output, or performance
  regression — in any language or stack. Also trigger when the user says "something's broken",
  "this doesn't work", "why is X happening", "help me debug", or pastes an error/stack trace.
  This skill enforces a hypothesis-verify-fix loop with a post-mortem, so never skip to fixing
  without going through the identification and verification steps first.
---

# Debug Skill

Debugging is not guessing. It is a structured process of forming hypotheses, testing them cheaply,
and narrowing the problem space until the root cause is confirmed — then fixing it.

Never jump to a fix without a verified cause. A fix without a confirmed root cause is just another guess.

---

## Phase 1: Triage — Collect Before Concluding

Before forming any hypothesis, gather the full picture.

**Checklist — ask or infer:**
- What is the exact symptom? (error message, wrong output, crash, hang, wrong behavior)
- Is there a stack trace, log output, or error code? → Read it carefully, top to bottom
- Is this reproducible? Always? Sometimes? Under specific conditions?
- When did it start? After a deploy, a dependency update, a config change, a code change?
- What has already been tried?
- What's the environment? (language, framework version, OS, platform, dev vs prod)

**Goal:** Separate *symptom* from *root cause*. The error message is almost never the root cause — it's the symptom. The root cause is what caused that error to occur.

Do not skip this phase even if the bug "seems obvious." Obvious bugs are where the worst assumptions live.

---

## Phase 2: Hypothesis Formation

State 1–3 ranked hypotheses. Be explicit.

**Format per hypothesis:**
```
H1 [confidence: high/medium/low] — <one sentence root cause>
  Reasoning: why you think this
  Evidence for: what supports it
  Evidence against: what argues against it
```

**Good hypotheses are:**
- Specific — not "something is wrong with the database" but "the query is returning stale data because the cache is not being invalidated on write"
- Falsifiable — you can write a test that would prove or disprove it
- Ranked — order by likelihood, not severity

If you don't have enough information to form a confident hypothesis, say so and go to Phase 3 (Gather) first.

---

## Phase 3: Verify — Write a Targeted Probe

**The rule:** Write the *smallest possible* code that tests the hypothesis. Not the fix. A probe.

> For language-specific tooling to run your probe (debugger setup, `EXPLAIN ANALYZE`, React DevTools, curl, memory snapshots), see forge's `references/debugging.md`.

A probe might be:
- A `console.log` / `print` at a specific point
- An assertion that checks a value you assume to be true
- A minimal reproduction case stripped of all unrelated code
- A direct test of the suspected function/module in isolation
- A curl/request to test a network assumption

**Probe requirements:**
- Target exactly one hypothesis
- Output should be a clear yes/no (or a value that confirms/denies)
- Should not modify any state or fix anything

Run the probe. Evaluate the output.

**If confirmed → go to Phase 5 (Fix)**
**If denied → go to Phase 4 (Narrow Down)**

---

## Phase 4: Narrow Down — When the Hypothesis is Wrong

Don't abandon structure just because you were wrong. Being wrong narrows the search space.

1. **Log what the probe revealed** — what is now *ruled out*
2. **Re-examine the triage data** — was there a signal you dismissed or didn't weight enough?
3. **Form the next hypothesis** — use what the failed probe taught you. The failure is data.
4. If you need more context to hypothesize, write a *gather probe* first:
   - A probe whose purpose is to expose state, not test a hypothesis
   - Examples: dump the full object, log the execution path, add a time check, inspect the call stack
5. Return to Phase 2 with the new information.

**The narrowing loop:**
```
Probe → Wrong → What did I learn? → New hypothesis → Probe → ...
```

Track each iteration. Don't re-test a ruled-out hypothesis.

---

## Phase 5: Fix — After Root Cause is Confirmed

Now and only now: write the fix.

**Fix principles:**
- **Minimal blast radius** — change only what is necessary to fix the confirmed root cause
- **Don't fix adjacent issues** in the same commit unless they're directly related
- **Explain the fix** — one sentence: "This fixes X by doing Y"
- Preserve the probe/test code as a regression test if possible

---

## Phase 6: Verify the Fix

Do not assume the fix worked. Confirm it.

- Re-run the original failing case → should now pass
- Run any adjacent test cases that could have been affected
- If the bug was intermittent: test the conditions that triggered it
- Check that no new symptoms appeared (regression check)

---

## Phase 7: Post-Mortem Note

Write a brief retrospective, especially when Phase 4 was triggered (your first hypothesis was wrong).

**Format:**
```
POST-MORTEM
-----------
Initial hypothesis: <what you thought it was>
Actual root cause:  <what it actually was>

Why I was wrong:
  <What assumption failed? What signal did you miss or misread?>

What I should have caught earlier:
  <Was there evidence in the triage data that pointed to the real cause?
   Was there a question you didn't ask? A log line you didn't read?
   A layer of the stack you didn't consider?>

What to watch for next time:
  <One or two heuristics for similar bugs in the future>
```

This is not self-punishment. It's calibration. The goal is to shorten the next debugging loop.

---

## Phase 8: Summary Note (PR-style)

Write a brief summary the user can use to understand or communicate what happened.

**Format:**
```
SUMMARY
-------
Bug:   <one sentence — what was broken and how it manifested>
Cause: <one sentence — confirmed root cause>
Fix:   <one sentence — what changed and why it solves it>
Files: <list of files changed, if known>
Risk:  <low/medium/high — and why>
```

Keep it under 8 lines. This is not a changelog — it's a quick handoff note.

---

## Principles to Never Violate

| Principle | Why |
|---|---|
| No fix without confirmed cause | Fixes based on guesses create new bugs and hide the original |
| Probe ≠ Fix | A probe that accidentally fixes things obscures causality |
| Symptom ≠ Root cause | The error line is where the program *noticed* the problem, not where it started |
| Log what you ruled out | Prevents circular debugging and losing track of progress |
| Post-mortem is not optional | The pattern that fooled you once will fool you again without reflection |

---

## Quick Reference: Common Misidentification Traps

- **Fixating on the error line** — the error occurs where the symptom surfaces, not where the cause lives
- **Assuming the last change caused it** — correlation is not causation; recent changes are suspects, not convicts
- **Skipping the logs** — the answer is usually already in the logs; read them fully before hypothesizing
- **Over-scoping the probe** — probes that test multiple things at once give ambiguous results
- **Trusting the happy path** — bugs usually live in the edge case you didn't think about
- **Not checking the environment** — "works on my machine" is usually an environment delta, not a code bug
- **Assuming the framework/library is correct** — dependencies have bugs too; check versions and changelogs
