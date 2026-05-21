# Forge & Debug — Software Engineering Skills

A collection of AI coding skills that enforce professional-grade code quality and systematic debugging workflows.

## Skills

### Forge — Elite Software Engineering

Forge transforms AI coding assistants into principal-level software engineers. Every line of code produced follows a rigorous 12-phase workflow covering the full software development lifecycle.

#### Phases

| Phase | Focus |
|---|---|
| 1 | Understand Before Building |
| 2 | Write Code That Lasts |
| 3 | Domain Standards (Backend, Frontend, Security, Architecture, Performance) |
| 4 | Systematic Debugging |
| 5 | Testing |
| 6 | Documentation |
| 7 | Git & Collaboration |
| 8 | Dependencies & Technical Debt |
| 9 | CI/CD & Deployment |
| 10 | Observability |
| 11 | Search & Research |
| 12 | Code Review |

#### Forge Reference Files

| File | Description |
|---|---|
| [patterns.md](skills/forge/references/patterns.md) | Architecture decisions and design patterns |
| [security-checklist.md](skills/forge/references/security-checklist.md) | Security best practices and checklists |
| [performance-tips.md](skills/forge/references/performance-tips.md) | Language/framework-specific performance tips |
| [debugging.md](skills/forge/references/debugging.md) | Debugger setup and language-specific tools |
| [testing.md](skills/forge/references/testing.md) | Mocking patterns and framework-specific testing |
| [database.md](skills/forge/references/database.md) | Schema design, indexing, query optimization |
| [api-design.md](skills/forge/references/api-design.md) | REST conventions, API design patterns |

### Debug — Structured Debugging

Debug enforces a hypothesis-verify-fix loop for methodically diagnosing and fixing bugs. It prevents jumping to fixes without confirmed root causes.

#### Phases

| Phase | Focus |
|---|---|
| 1 | Triage — Collect symptoms, environment, reproduction steps |
| 2 | Hypothesis Formation — Rank 1-3 specific, falsifiable hypotheses |
| 3 | Verify — Write targeted probes to test hypotheses |
| 4 | Narrow Down — Iterate when hypotheses are wrong |
| 5 | Fix — Minimal change after root cause is confirmed |
| 6 | Verify the Fix — Confirm the original case passes |
| 7 | Post-Mortem — Document why initial assumptions were wrong |
| 8 | Summary — Brief handoff note for PRs or communication |

#### Core Principles

- No fix without confirmed cause
- Probe is not the same as fix
- Symptom is not the same as root cause
- Log what you ruled out to prevent circular debugging
- Post-mortem is not optional — it calibrates future debugging

## Usage

Forge is designed for use with [OpenCode](https://opencode.ai) and compatible AI coding assistants. Place the `skills/` directory in your project's `.opencode/` folder or configure it in your skills path.

## License

See individual files for license information.
