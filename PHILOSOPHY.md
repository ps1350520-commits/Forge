# The Philosophy of Forge

Why 12 phases? Why this order? What breaks when you skip them?

---

## Why 12 Phases?

Twelve is not arbitrary. It maps to the complete lifecycle of professional software engineering:

| Phases | Purpose |
|---|---|
| 1-2 | **Build it right** — understand the problem, write clean code |
| 3 | **Apply domain knowledge** — security, architecture, performance |
| 4 | **When things break** — systematic debugging |
| 5-6 | **Prove and explain it** — tests and documentation |
| 7-8 | **Ship and maintain it** — git, dependencies, debt |
| 9-10 | **Operate it** — CI/CD, observability |
| 11-12 | **Improve it** — research, review, iterate |

Each phase addresses a dimension of quality that, if ignored, becomes a production incident.

---

## Why This Order?

The phases are ordered by **cost of failure**. The earlier a phase is skipped, the more expensive the consequences.

### Phase 1 must come before Phase 2

If you write code before understanding the problem, you build the wrong thing correctly. This is the most expensive mistake in software engineering. A feature built perfectly to the wrong spec costs more than no feature at all — it has to be discovered, undone, and rebuilt.

**What breaks if you skip Phase 1:** You ship features nobody needs. You build solutions to problems that don't exist. You miss constraints that invalidate your entire approach.

### Phase 2 must come before Phase 3

Clean code is the foundation. Domain standards (security, architecture, performance) are applied on top of that foundation. If the code is already messy, applying domain standards becomes impossible — you can't secure a system you can't read.

**What breaks if you skip Phase 2:** Technical debt accumulates from day one. Every future change costs more. Security reviews fail because the code is too complex to audit.

### Phase 3 must come before Phase 4

You can't debug what you haven't built to standards. Phase 3 ensures the code has health endpoints, structured logging, and proper error handling — all of which make Phase 4 (debugging) possible. Code without observability is un-debuggable.

**What breaks if you skip Phase 3:** You have no way to diagnose issues in production. No logs, no health checks, no error context. Debugging becomes guesswork.

### Phase 4 is conditional

Phase 4 (debugging) only activates when something breaks. It's always available but not always used. Its position after Phase 3 is intentional — you need domain standards in place before debugging is effective.

### Phase 5 must come before Phase 6

You can't document behavior that isn't verified. Tests define what the code is supposed to do. Documentation explains why it does it. If you write docs before tests, you're documenting assumptions, not facts.

**What breaks if you skip Phase 5:** You have no proof the code works. Refactoring becomes terrifying. Every change is a potential regression. You ship bugs that tests would have caught.

### Phase 6 must come before Phase 7

Undocumented code cannot be reviewed effectively. Phase 6 ensures the code has JSDoc, env var documentation, and architectural decision records. Phase 7 (git collaboration) then captures these decisions in commit messages and PR descriptions.

**What breaks if you skip Phase 6:** Future engineers (including you) don't know why decisions were made. Onboarding takes weeks instead of days. The same mistakes are repeated.

### Phase 7 must come before Phase 8

Clean git history and collaboration practices make dependency management and debt tracking possible. If commits are meaningless, you can't trace when a bad dependency was introduced or when debt was incurred.

**What breaks if you skip Phase 7:** You can't bisect to find when a bug was introduced. PRs are unreviewable. Team coordination breaks down.

### Phase 8 must come before Phase 9

You can't build a CI/CD pipeline for a project with untracked dependencies and invisible technical debt. Phase 8 ensures you know what you're shipping and what risks it carries.

**What breaks if you skip Phase 8:** Vulnerable dependencies ship to production. Technical debt compounds silently. The cost of change grows exponentially.

### Phase 9 must come before Phase 10

CI/CD deploys the code. Observability monitors it. You can't monitor what isn't deployed, and you can't trust a deployment pipeline that doesn't gate on observability requirements.

**What breaks if you skip Phase 9:** Manual deploys are inconsistent. Rollbacks are slow or impossible. No one knows if the deploy succeeded.

### Phase 10 must come before Phase 11

Observability tells you what's wrong in production. Research tells you how to fix it or improve it. You can't research the right solution without knowing what problem you're actually solving.

**What breaks if you skip Phase 10:** You're flying blind. Performance regressions go unnoticed. Errors accumulate silently. You optimize the wrong things.

### Phase 11 must come before Phase 12

Research informs review. You can't effectively review code against best practices you haven't researched. Phase 11 ensures you're comparing against current knowledge, not outdated assumptions.

**What breaks if you skip Phase 11:** You reinvent solved problems. You use outdated patterns. You miss security advisories. You make claims without benchmarks.

### Phase 12 closes the loop

Code review is the final gate. It checks everything: correctness (Phase 1-2), standards (Phase 3), testability (Phase 5), documentation (Phase 6), consistency (Phase 7), dependencies (Phase 8), deployability (Phase 9), observability (Phase 10).

**What breaks if you skip Phase 12:** Bugs reach production. Security issues ship. Technical debt goes unchallenged. Team standards erode.

---

## The Dependency Graph

```
Phase 1 (Understand)
    ↓
Phase 2 (Write Code)
    ↓
Phase 3 (Domain Standards)
    ↓
Phase 4 (Debug) ← conditional
    ↓
Phase 5 (Test)
    ↓
Phase 6 (Document)
    ↓
Phase 7 (Git & Collaborate)
    ↓
Phase 8 (Dependencies & Debt)
    ↓
Phase 9 (CI/CD)
    ↓
Phase 10 (Observability)
    ↓
Phase 11 (Research)
    ↓
Phase 12 (Review)
    ↻
    Loops back: review findings feed into the next Phase 1
```

---

## What If You Only Have Time for 3 Phases?

If you're in a rush and can only enforce a subset, these three give the most leverage:

1. **Phase 1 (Understand)** — Prevents building the wrong thing
2. **Phase 5 (Test)** — Proves the thing works
3. **Phase 12 (Review)** — Catches what you missed

But this is a compromise, not a recommendation. The full 12-phase workflow is designed so that each phase reduces the workload of the next. Skipping phases doesn't save time — it defers cost.

---

## Why Not Fewer Phases?

You could merge phases. Testing and Documentation could be one phase. Git and Dependencies could be one phase. But splitting them serves a purpose:

- **Each phase has a clear entry and exit condition.** You know when you're done with Phase 1 and ready for Phase 2.
- **Each phase can be invoked independently.** "Use Forge, focus on testing" activates Phase 5 without running the full workflow.
- **Each phase has its own reference files.** Phase 5 loads `references/testing.md`. Phase 3 loads `references/security-checklist.md`. Merging phases would blur these boundaries.

---

## The Debug Skill's Relationship to Forge

Debug is not a separate philosophy — it's Phase 4 of Forge, extracted into its own skill because debugging deserves its own depth.

When you use Debug standalone, you're using the same methodology that Forge uses in Phase 4, but with more detail:
- Hypothesis formation with confidence ranking
- Probe discipline (probe ≠ fix)
- The narrowing loop
- Post-mortem calibration

When you use Forge, Debug's methodology is available whenever something breaks during any phase.

---

## The Core Belief

Forge exists because of one belief:

> **The cost of software is not in writing it. The cost is in maintaining it, debugging it, extending it, and explaining it to the next engineer.**

Every phase of Forge is designed to reduce that cost. Not the cost of writing the first version — the cost of the thousand changes that come after.

If you skip phases, you're not saving time. You're borrowing from the future at compound interest.
