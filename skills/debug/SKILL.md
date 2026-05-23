---
name: debug
description: >
  Structured debugging discipline combining hypothesis-verify-fix rigor with mantra-based
  constraints. Trigger whenever the user reports a bug, unexpected behavior, error, crash,
  wrong output, or performance regression — in any language or stack. Also trigger when the
  user says "something's broken", "this doesn't work", "why is X happening", "help me debug",
  or pastes an error/stack trace. Enforces: reproduce first, escalating investigation, falsify
  before committing, breadcrumb ledger across every run, post-mortem on every non-trivial fix.
  Never skip to fixing without completing identification and verification.
---

# Debug — Structured Debugging Discipline

Debugging is not guessing. It is a structured process of reproducing, tracing, hypothesizing,
falsifying, and fixing — in that order. A fix without a confirmed root cause is just another
guess that will cost someone later.

---

## Mantra — Recite Verbatim at Session Start

> **Mantra:**
> 1. **First is reproducibility.** Can the issue be reproduced reliably?
> 2. **Know the fail path.** Debugger first; then source trace + knob enumeration; then in-code instrumentation.
> 3. **Question your hypothesis.** What would disprove it?
> 4. **Every run is a breadcrumb.** Cross-reference all of them.

Recite this block **once**, verbatim, as the first thing in your first response of any debug
session. Then begin work. If the user says "skip the mantra" → skip the recital but still
apply the four steps silently. Never paraphrase, shorten, or skip lines.

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

**Goal:** Separate *symptom* from *root cause*. The error message is almost never the root
cause — it's the symptom. The root cause is what caused that error to occur.

Do not skip this phase even if the bug "seems obvious." Obvious bugs are where the worst
assumptions live.

---

## Phase 2: Reproduce Reliably

Build a runnable repro before anything else. This is **non-negotiable**.

- **Reliable repro** → capture the exact steps, inputs, and environment as a runnable
  artifact: failing test, curl script, CLI invocation, replay harness.
- **Flaky repro** → the bug is not yet fully debuggable. Raise the rate first: loop the
  trigger, parallelize, add stress, narrow timing windows, inject sleeps. 50% flake is
  debuggable; 1% is not.
- **No repro at all** → **stop**. Say so explicitly. Ask the user for env access, captured
  artifacts (HAR, log dump, core dump), or permission to instrument. Do **not** proceed to
  hypothesize.

**Target:** a fast (1–5 s), deterministic pass/fail signal. Pin time, seed the RNG, freeze
network, isolate filesystem.

**If you catch yourself proposing a fix without a reliable repro, stop and return to this phase.**

---

## Phase 3: Know the Fail Path — Escalating Investigation

Once reproducible, find *where* the code breaks and *what stops it from breaking*. The
differential narrows the search. Try in this order — escalate only when the prior tactic fails.

### Tier 1: Attach a Debugger
If the env supports it, attach and step to the failure site. One breakpoint beats ten logs.
Do this **before** turning any knobs.

### Tier 2: Source Trace + Knob Enumeration
If no debugger (or it can't reach the bug), trace the code path end-to-end and list every
knob that can influence the outcome:
- Config flags, env vars, feature toggles
- Branch conditions, input shape
- Timing, concurrency, build options

Each knob is a candidate axis to flip in the differential. **Flip one at a time.**

### Tier 3: In-Code Instrumentation
If outside knobs can't move the failure, go inside: `printf` / log statements at the
suspected fail site, dump the relevant internal state. **Tag every probe with a unique
prefix** (e.g. `[DBG-a4f2]`) so cleanup is a single grep. Let the trace show where
reality diverges from your model.

### Tier 4: Bisection
When the failure site is still unclear, divide the code path in half. Is the bug in the
first half or the second? Repeat. You'll find it in O(log n) steps. Use `git bisect` for
regression hunting across commits.

> For language-specific tooling to run your probes (debugger setup, `EXPLAIN ANALYZE`,
> React DevTools, curl, memory snapshots), see forge's `references/debugging.md`.

---

## Phase 4: Hypothesis Formation

State 1–3 ranked hypotheses. Be explicit. **Generate multiple hypotheses, not one.**
Single-hypothesis thinking anchors on the first plausible idea.

**Format per hypothesis:**
```
H1 [confidence: high/medium/low] — <one sentence root cause>
  Reasoning: why you think this
  Evidence for: what supports it
  Evidence against: what argues against it
  Disproof: what single experiment would kill this hypothesis
```

**Good hypotheses are:**
- Specific — not "something is wrong with the database" but "the query is returning stale
  data because the cache is not being invalidated on write"
- Falsifiable — you can write a test that would prove or disprove it
- Ranked — order by likelihood, not severity
- End-to-end — the hypothesis must explain the symptom completely, not just partially

If you don't have enough information to form a confident hypothesis, say so and return to
Phase 3 to gather more data.

---

## Phase 5: Falsify — Disprove Before You Prove

**The core discipline:** When a candidate root cause surfaces, try to **kill it before
confirming it**.

1. For each hypothesis, identify the simplest **proof** and the cleanest **disproof**
2. **Run the disproof first.** If the hypothesis survives, it's real. If it dies, you
   saved yourself from chasing a phantom.
3. Write the *smallest possible* code that tests one hypothesis. Not the fix. A probe.

**Probe requirements:**
- Target exactly one hypothesis
- Output should be a clear yes/no (or a value that confirms/denies)
- Should not modify any state or fix anything
- A probe that accidentally fixes things obscures causality — that's not a probe, it's
  a gamble

**If confirmed → go to Phase 7 (Fix)**
**If denied → go to Phase 6 (Narrow Down)**

---

## Phase 6: Narrow Down — When the Hypothesis is Wrong

Don't abandon structure just because you were wrong. Being wrong narrows the search space.
The failure is data.

1. **Log what the probe revealed** — what is now *ruled out*. Update the ledger (Phase 9).
2. **Re-examine the triage data** — was there a signal you dismissed or didn't weight enough?
3. **Walk the breadcrumb ledger** — does the new information hold for *every* prior
   observation, not just the most recent?
4. **Form the next hypothesis** — use what the failed probe taught you.
5. If you need more context, write a *gather probe* first:
   - A probe whose purpose is to expose state, not test a hypothesis
   - Examples: dump the full object, log the execution path, add a time check, inspect the
     call stack
6. Return to Phase 4 with the new information.

**The narrowing loop:**
```
Probe → Wrong → What did I learn? → Check ledger → New hypothesis → Probe → ...
```

Track each iteration. Don't re-test a ruled-out hypothesis.

---

## Phase 7: Fix — After Root Cause is Confirmed

Now and only now: write the fix.

**Fix principles:**
- **Minimal blast radius** — change only what is necessary to fix the confirmed root cause
- **Don't fix adjacent issues** in the same commit unless they're directly related
- **Explain the fix** — one sentence: "This fixes X by doing Y"
- Preserve the probe/test code as a regression test if possible
- **Make the failure loud** — if possible, add a check so this class of bug fails fast
  and visibly in the future instead of silently corrupting state

---

## Phase 8: Verify the Fix

Do not assume the fix worked. Confirm it.

- Re-run the original failing repro → should now pass
- Run any adjacent test cases that could have been affected
- If the bug was intermittent: test the conditions that triggered it, at elevated stress
- Check that no new symptoms appeared (regression check)
- Write a regression test that would have caught this bug — the fix isn't complete until
  the test exists

---

## Phase 9: Breadcrumb Ledger — Your Memory Across the Session

Maintain a running **ledger** of every experiment in this session. Each entry:

```
Run #N: <what changed> → <what happened> → <ruled in/out>
```

**Ledger discipline:**
- When a new hypothesis surfaces, walk the ledger. Does it hold for **every** prior
  observation, not just the most recent?
- If any past run contradicts it, the hypothesis is wrong or incomplete — refine or discard.
- When in doubt, design the **single experiment** whose outcome makes it certain. Run that
  next, instead of churning on adjacent runs.
- Update the ledger after every run. It is your memory across the session.

---

## Phase 10: Post-Mortem

Write a brief retrospective, especially when Phase 6 was triggered (your first hypothesis
was wrong). Skip only for trivial one-liner fixes where the PR description is the record.

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

Why it slipped through:
  <CI gap? Latent code? Workload gap? Incomplete prior fix? Review miss?>

What to watch for next time:
  <One or two heuristics for similar bugs in the future>
```

This is not self-punishment. It's calibration. The goal is to shorten the next debugging loop.

---

## Phase 11: Summary Note (PR-style)

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

## Bug Taxonomy — Quick Classification

| Category | Signature | First move |
|---|---|---|
| Logic error | Wrong output for valid input | Trace data transformations step-by-step |
| Null/type error | Crash on undefined, type mismatch | Trace the value's origin; enable strict mode |
| Async/race condition | Intermittent, order-dependent | Log timestamps; look for missing `await` |
| State bug (FE) | Stale data, wrong render | Log state before/after; check mutation vs replace |
| Integration bug | Works in isolation, fails together | Inspect raw HTTP; compare dev vs prod env vars |
| Performance bug | Slow, memory growth | Profile first; check N+1, bundle size, heap |
| Config/env bug | Works locally, fails in CI/prod | Diff env vars; check deployed artifact is current |
| Flaky test | Passes/fails non-deterministically | See Flaky Bug Playbook below |

---

## Flaky Bug Playbook

Flaky bugs deserve their own protocol because they violate Phase 2 (reliable repro).

1. **Measure the flake rate.** Run the failing case N times (N ≥ 20). What's the failure %?
2. **Classify the flake source:**
   - **Timing/ordering** — add sleeps, reorder operations, force serial execution
   - **Shared mutable state** — run in isolation. Does it still flake?
   - **External dependency** — network, filesystem, clock. Mock or pin it.
   - **Resource exhaustion** — memory, file handles, ports. Monitor during the run.
3. **Amplify the flake.** Make it fail 50%+ before debugging:
   - Loop the trigger in a tight loop
   - Add concurrent callers / threads
   - Reduce timeouts
   - Add artificial jitter (`sleep(random(0, 50))`)
4. **Once at 50%+**, it's a regular bug. Return to Phase 3.

---

## Severity-Based Time-Boxing

Not all bugs deserve infinite investigation time. Set expectations early:

| Severity | Time box | Escalation |
|---|---|---|
| P0 — Production down | 30 min investigation, then escalate | Bring in additional engineers, check rollback |
| P1 — Major feature broken | 2 hours | Step back and re-triage from scratch |
| P2 — Minor bug | 4 hours | Consider workaround + backlog ticket |
| P3 — Cosmetic/edge case | 1 hour | Fix or won't-fix decision |

If you hit the time box: stop, summarize what you know, what you've ruled out, and what
the most promising next step is. Don't keep churning in silence.

---

## Anti-Pattern Gallery — Debugging Smells

These are signs you've left the disciplined path:

- **Shotgun debugging** — changing multiple things at once "to see if it helps." You won't
  know which change fixed it, or if any did.
- **Fix-first debugging** — writing a fix before confirming the root cause. You're guessing.
- **Recency bias** — "it must be the last commit." Recent changes are suspects, not convicts.
  Correlation is not causation.
- **Fixating on the error line** — the error occurs where the symptom surfaces, not where
  the cause lives. Read the *bottom* of the stack trace first.
- **Skipping the logs** — the answer is usually already in the logs. Read them fully before
  hypothesizing.
- **Over-scoping the probe** — probes that test multiple things at once give ambiguous results.
  One probe, one hypothesis.
- **Trusting the happy path** — bugs live in edge cases you didn't think about.
- **Ignoring the environment** — "works on my machine" is usually an environment delta.
- **Assuming dependencies are correct** — libraries have bugs too. Check versions and changelogs.
- **Circular debugging** — re-testing hypotheses you already ruled out. Check the ledger.
- **Comfort-zone debugging** — only looking in the layer you know best. The bug is often at
  the boundary between layers.

---

## Principles to Never Violate

| Principle | Why |
|---|---|
| No fix without confirmed cause | Fixes based on guesses create new bugs and hide the original |
| No hypothesis without repro | You can't reason about what you can't observe |
| Probe ≠ Fix | A probe that accidentally fixes things obscures causality |
| Symptom ≠ Root cause | The error line is where the program *noticed* the problem, not where it started |
| Disprove before you prove | Confirmation bias will anchor you on the first plausible idea |
| Log what you ruled out | Prevents circular debugging and losing track of progress |
| Every run updates the ledger | The ledger is your memory; without it you're starting over each time |
| Post-mortem is not optional | The pattern that fooled you once will fool you again without reflection |

---

## Operating Rules

- Recite the mantra block **once** per debug session, in your first response.
- Apply all phases **in order**:
  - Do not propose a fix before Phase 2 is satisfied (reliable repro exists).
  - Do not start testing hypotheses before Phase 3 has narrowed the fail path.
  - Do not commit to a hypothesis before Phase 5 has tried to disprove it.
  - Do not declare a hypothesis correct until the ledger confirms it against every prior run.
- The mantra and phases are constraints **you** carry through the session — not advice to
  deliver back to the user.
- If the user asks you to "just fix it" — explain the risk, but if they insist after one
  explanation, comply with a clear caveat about confidence level.
