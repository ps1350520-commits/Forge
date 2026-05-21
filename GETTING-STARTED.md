# Getting Started with Forge & Debug

This guide walks you through installing, configuring, and using the Forge and Debug skills with your AI coding assistant.

---

## 1. Installation

### Option A: Per-Project (Recommended)

Copy the `skills/` directory into your project's `.opencode/` folder:

```bash
# From the root of your project
mkdir -p .opencode
cp -r /path/to/Forge/skills .opencode/
```

Your project structure should look like:

```
your-project/
  .opencode/
    skills/
      forge/
        SKILL.md
        references/
          patterns.md
          security-checklist.md
          performance-tips.md
          debugging.md
          testing.md
          database.md
          api-design.md
      debug/
        SKILL.md
  src/
  tests/
  package.json
```

### Option B: Global Installation

Copy to your global OpenCode config so all projects can use the skills:

```bash
cp -r /path/to/Forge/skills ~/.config/opencode/
# or on Windows:
xcopy /E /I /path/to/Forge/skills %USERPROFILE%\.config\opencode\skills\
```

### Option C: Clone Directly

```bash
git clone https://github.com/ps1350520-commits/Forge.git
cd Forge
cp -r skills /path/to/your-project/.opencode/
```

---

## 2. Verify Installation

Open your AI coding assistant and ask:

```
"What skills do you have available?"
```

You should see **forge** and **debug** listed. If not, double-check the directory structure.

---

## 3. Using Forge During a Coding Task

Forge activates automatically when you ask it to. Here's how it works in practice:

### Simple Task (< 50 lines)

```
"Use Forge to write a function that validates email addresses"
```

Forge will:
1. State its approach in 1-2 sentences
2. Note tradeoffs (regex vs library)
3. Write the code with proper naming, types, and error handling

### Complex Task (multi-file feature)

```
"Use Forge to add user registration with email verification"
```

Forge will:
1. **Phase 1:** Restate the goal, identify constraints, map data flow, find risks, outline structure
2. **Phase 2:** Write clean code with proper naming, error handling, and type safety
3. **Phase 3:** Apply domain standards (validation at boundary, layer separation, security checks)
4. **Phase 5:** Write tests for happy path, error paths, and edge cases
5. **Phase 6:** Add JSDoc/docstrings to public functions
6. **Phase 12:** Self-review against the checklist

### Code Review

```
"Use Forge to review this pull request"
```

Forge will check: correctness, completeness, security, performance, testability, readability, consistency — and give feedback in the Blocker/Suggestion/Discussion/Praise format.

---

## 4. Using Debug When Investigating a Bug

Debug activates when you report a problem:

```
"Use Debug — my API returns 403 for admin users"
```

### What Debug Does

**Step 1 — Triage:** Asks for the exact symptom, environment, reproduction steps, and what's already been tried.

**Step 2 — Hypothesize:** Forms 1-3 ranked hypotheses with confidence levels and reasoning.

**Step 3 — Probe:** Writes the smallest possible code change to test the hypothesis (not the fix).

**Step 4 — Iterate:** If the hypothesis is wrong, uses what was learned to form a new one.

**Step 5 — Fix:** Only after the root cause is confirmed, writes the minimal fix.

**Step 6 — Verify:** Confirms the original failing case now passes.

**Step 7 — Post-Mortem:** Documents why initial assumptions were wrong (if any were).

**Step 8 — Summary:** Gives you a brief note you can use in a PR or standup.

### Example Debug Session

```
You: "Use Debug — my payment processing works locally but fails in production with a timeout"

Debug:
  Triage: When does it fail? Always or intermittently? What changed recently?
  You: "Always fails since yesterday's deploy. We updated the payment SDK."

  H1 [high]: New SDK version has different timeout defaults
  H2 [medium]: Production firewall blocks the new SDK's endpoint
  H3 [low]: Production env var for payment gateway URL is stale

  Probe: Log the actual timeout value and endpoint URL in production...
  Result: Timeout is 500ms (was 5000ms). SDK default changed in v3.

  Fix: Set timeout explicitly: new PaymentSDK({ timeout: 5000 })
  Verified: Payment processes successfully.
```

---

## 5. When to Use Each Skill

| Situation | Skill |
|---|---|
| Writing new code | Forge |
| Refactoring existing code | Forge |
| Reviewing a PR | Forge |
| Designing an API | Forge + `references/api-design.md` |
| Something is broken | Debug |
| Intermittent failures | Debug |
| Performance regression | Debug + `references/performance-tips.md` |
| Security audit | Forge + `references/security-checklist.md` |
| Writing tests | Forge + `references/testing.md` |
| Database optimization | Forge + `references/database.md` |

---

## 6. Tips for Best Results

1. **Be specific about constraints.** "Use Forge to build this in Python with FastAPI" produces better output than "build this."

2. **Don't skip the triage in Debug.** If Debug asks questions, answer them. The quality of the diagnosis depends on the quality of the input.

3. **Let Forge run its phases.** You can ask it to focus on specific phases ("Use Forge, focus on testing and documentation"), but the full workflow produces the best results.

4. **Reference files are loaded automatically** when the task requires them. You don't need to request them explicitly.

5. **Forge and Debug complement each other.** Use Forge to build, Debug to fix. They share the debugging reference files.
