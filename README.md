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

### Debug — Structured Debugging Discipline

A comprehensive debugging methodology combining hypothesis-verify-fix rigor with mantra-based constraints, falsification-first methodology, and breadcrumb ledger tracking.

#### Mantra

> 1. **First is reproducibility.** Can the issue be reproduced reliably?
> 2. **Know the fail path.** Debugger first; then source trace + knob enumeration; then in-code instrumentation.
> 3. **Question your hypothesis.** What would disprove it?
> 4. **Every run is a breadcrumb.** Cross-reference all of them.

#### Phases

| Phase | Focus |
|---|---|
| 1 | Triage — Collect symptoms, environment, reproduction steps |
| 2 | Reproduce Reliably — Build a fast, deterministic pass/fail signal |
| 3 | Know the Fail Path — Debugger, source trace, instrumentation, bisection |
| 4 | Hypothesis Formation — Rank 1-3 specific, falsifiable hypotheses |
| 5 | Falsify — Disprove before you prove |
| 6 | Narrow Down — Iterate when hypotheses are wrong, update ledger |
| 7 | Fix — Minimal change after root cause is confirmed |
| 8 | Verify the Fix — Confirm the original case passes, add regression test |
| 9 | Breadcrumb Ledger — Running log of every experiment in the session |
| 10 | Post-Mortem — Document why initial assumptions were wrong |
| 11 | Summary Note — Brief handoff note for PRs or communication |

#### Core Principles

- No fix without confirmed cause
- No hypothesis without repro
- Probe is not the same as fix
- Symptom is not the same as root cause
- Disprove before you prove
- Log what you ruled out to prevent circular debugging
- Every run updates the ledger
- Post-mortem is not optional — it calibrates future debugging

---

## Getting Started

See [GETTING-STARTED.md](GETTING-STARTED.md) for step-by-step setup and walkthroughs.

**Quick start:**
1. Copy `skills/` into your project's `.opencode/` folder
2. Open your AI coding assistant and invoke Forge on any coding task
3. Invoke Debug when a bug appears

## Philosophy

See [PHILOSOPHY.md](PHILOSOPHY.md) for why Forge has 12 phases, why they're ordered this way, and what breaks when you skip them.

---

## Usage

### Setup Steps

1. **Clone or download** this repository
2. **Copy the skills folder** into your project:
   ```bash
   # Option A: Copy into a specific project
   cp -r skills /path/to/your-project/.opencode/

   # Option B: Copy to global OpenCode config
   cp -r skills ~/.config/opencode/
   ```
3. **Verify** your AI coding assistant detects the skills. They should appear when you ask it to list available skills.

### Invoking Forge

Ask your AI assistant to use Forge on any coding task:

```
"Use Forge to build a user authentication endpoint"
"Use Forge to review this PR"
"Use Forge to refactor this module"
```

Forge will run through its 12 phases automatically — understanding the goal, writing clean code, applying domain standards, adding tests, documenting, and more.

### Invoking Debug

When something breaks:

```
"Use Debug to figure out why this endpoint returns 500"
"Use Debug — my tests are failing intermittently"
"Use Debug — why is this query so slow?"
```

Debug will triage, form hypotheses, write probes, confirm the root cause, then fix — never guessing.

---

## Examples

### Example 1: Building a Python API with Forge (12 Phases)

**Request:** "Use Forge to build a REST API for managing a todo list with Python and FastAPI."

Forge would proceed through all 12 phases:

**Phase 1 — Understand:** Goal is a CRUD API for todos. Constraints: Python, FastAPI, SQLite. Risks: input validation, SQL injection. Structure: `main.py`, `models.py`, `schemas.py`, `routes.py`.

**Phase 2 — Write Code:** Clean naming, early returns, proper error types.

```python
# schemas.py
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional

class TodoCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    description: Optional[str] = None
    due_date: Optional[datetime] = None

class TodoResponse(BaseModel):
    id: int
    title: str
    description: Optional[str]
    due_date: Optional[datetime]
    completed: bool
    created_at: datetime

# models.py
from sqlmodel import SQLModel, Field

class Todo(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    title: str = Field(max_length=200)
    description: str | None = None
    due_date: datetime | None = None
    completed: bool = False
    created_at: datetime = Field(default_factory=datetime.utcnow)

# routes.py
from fastapi import APIRouter, HTTPException, status
from sqlmodel import Session, select

router = APIRouter(prefix="/todos", tags=["todos"])

@router.post("/", response_model=TodoResponse, status_code=status.HTTP_201_CREATED)
def create_todo(todo: TodoCreate, session: Session = Depends(get_session)):
    db_todo = Todo(**todo.model_dump())
    session.add(db_todo)
    session.commit()
    session.refresh(db_todo)
    return db_todo

@router.get("/", response_model=list[TodoResponse])
def list_todos(skip: int = 0, limit: int = 20, session: Session = Depends(get_session)):
    return session.exec(select(Todo).offset(skip).limit(limit)).all()

@router.get("/{todo_id}", response_model=TodoResponse)
def get_todo(todo_id: int, session: Session = Depends(get_session)):
    todo = session.get(Todo, todo_id)
    if not todo:
        raise HTTPException(status_code=404, detail="Todo not found")
    return todo

@router.patch("/{todo_id}", response_model=TodoResponse)
def update_todo(todo_id: int, todo: TodoCreate, session: Session = Depends(get_session)):
    db_todo = session.get(Todo, todo_id)
    if not db_todo:
        raise HTTPException(status_code=404, detail="Todo not found")
    for key, value in todo.model_dump(exclude_unset=True).items():
        setattr(db_todo, key, value)
    session.add(db_todo)
    session.commit()
    session.refresh(db_todo)
    return db_todo

@router.delete("/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_todo(todo_id: int, session: Session = Depends(get_session)):
    todo = session.get(Todo, todo_id)
    if not todo:
        raise HTTPException(status_code=404, detail="Todo not found")
    session.delete(todo)
    session.commit()
```

**Phase 3 — Domain Standards:** Validation via Pydantic (done). Layer separation: schemas, models, routes. Health endpoint added.

**Phase 4 — Debugging:** Not needed for fresh code, but protocol is documented for future issues.

**Phase 5 — Testing:**

```python
# test_routes.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_todo_returns_201():
    res = client.post("/todos/", json={"title": "Buy milk"})
    assert res.status_code == 201
    assert res.json()["title"] == "Buy milk"
    assert res.json()["completed"] is False

def test_create_todo_empty_title_returns_422():
    res = client.post("/todos/", json={"title": ""})
    assert res.status_code == 422

def test_get_nonexistent_todo_returns_404():
    res = client.get("/todos/9999")
    assert res.status_code == 404

def test_list_todos_returns_empty_list():
    res = client.get("/todos/")
    assert res.status_code == 200
    assert res.json() == []
```

**Phase 6 — Documentation:** JSDoc/docstrings on public functions, env vars documented.

**Phase 7 — Git:** Conventional commit: `feat(todos): add CRUD API for todo management`

**Phase 8 — Dependencies:** FastAPI, SQLModel, Pydantic — all well-maintained, popular, MIT licensed.

**Phase 9 — CI/CD:** Lint (ruff), test (pytest), build, deploy pipeline.

**Phase 10 — Observability:** Structured logging, `/health` and `/ready` endpoints.

**Phase 11 — Research:** Checked FastAPI docs for latest patterns, SQLModel for ORM best practices.

**Phase 12 — Code Review:** Checklist applied — correctness, completeness, security, performance, testability, readability, consistency.

---

### Example 2: Diagnosing a Bug with Debug Methodology

**Request:** "Use Debug — my login endpoint returns 500 but only in production."

Debug would proceed through its 11 phases:

**Phase 1 — Triage:**
- Symptom: POST /auth/login returns 500 in production, works locally
- Environment: Production (Docker, PostgreSQL 15, Node 20) vs Dev (local Node 20, SQLite)
- When started: After last deploy (2 hours ago)
- Already tried: Restarted container — same error

**Phase 2 — Hypothesis Formation:**
```
H1 [confidence: high] — Database connection fails in production because the DATABASE_URL env var is misconfigured
  Reasoning: Works locally with SQLite, production uses PostgreSQL. Last deploy may have changed env vars.
  Evidence for: 500 errors on first DB interaction are commonly connection issues
  Evidence against: Other endpoints that use the DB work fine

H2 [confidence: medium] — Password hashing library behaves differently between environments
  Reasoning: bcrypt native bindings can differ between local and Docker builds
  Evidence for: bcrypt has caused platform-specific issues before
  Evidence against: No error about native modules in logs

H3 [confidence: low] — Rate limiter middleware throws on missing Redis in production
  Reasoning: Production uses Redis for rate limiting, dev does not
  Evidence for: Login endpoint is rate-limited
  Evidence against: Redis connection errors would show in logs
```

**Phase 3 — Verify (Probe):**
```python
# Minimal probe — test DB connection directly
import os
from app.db import get_session

session = get_session()
try:
    result = session.exec("SELECT 1")
    print("DB connection OK")
except Exception as e:
    print(f"DB connection FAILED: {e}")
```
Result: DB connection OK. H1 denied.

**Phase 4 — Narrow Down:**
H1 ruled out. DB works. Checking production logs more carefully — found: `bcrypt.error: password cannot be longer than 72 bytes`. The login form allows unlimited length locally but production has no client-side limit.

New hypothesis: `H2 [confidence: high] — Password longer than 72 bytes causes bcrypt to throw`

**Phase 5 — Fix:**
```python
# Add server-side validation limit
class LoginSchema(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8, max_length=72)  # bcrypt limit
```

**Phase 6 — Verify:**
- Tested with 73-char password → now returns 400 with validation error instead of 500
- Tested with 72-char password → works correctly
- Tested with normal password → works correctly

**Phase 7 — Post-Mortem:**
```
POST-MORTEM
-----------
Initial hypothesis: Database connection misconfiguration
Actual root cause:  bcrypt throws on passwords > 72 bytes, no server-side validation

Why I was wrong:
  Assumed the environment difference was the cause. Didn't check the actual error message
  in production logs until Phase 4. The bcrypt limit is documented but easy to forget.

What I should have caught earlier:
  The production error log literally said "password cannot be longer than 72 bytes".
  I should have read the full stack trace in Phase 1 instead of jumping to env var theories.

What to watch for next time:
  Always read the full error message before forming hypotheses.
  bcrypt's 72-byte limit applies to all password inputs — validate at the boundary.
```

**Phase 8 — Summary:**
```
SUMMARY
-------
Bug:   Login endpoint returns 500 for passwords longer than 72 characters
Cause: bcrypt library throws an unhandled error when input exceeds 72 bytes
Fix:   Added max_length=72 validation to login schema — returns 400 instead of 500
Files: schemas.py (LoginSchema.password field)
Risk:  low — adds validation, no behavior change for valid inputs
```

---

## License

See individual files for license information.
