# Testing Patterns & Strategies

## Test Structure — The AAA Pattern

Every test should follow Arrange → Act → Assert:

```typescript
it('returns discounted price for premium users', () => {
  // Arrange
  const price = 100
  const tier = 'premium'

  // Act
  const result = calculateDiscount(price, tier)

  // Assert
  expect(result).toBe(90)
})
```

Keep each section visually separated. If Arrange is longer than ~10 lines,
extract a factory or builder helper.

---

## Naming Conventions

```
describe('SubjectUnderTest')
  describe('methodOrScenario')
    it('does X when Y')
    it('throws Z when input is invalid')
    it('returns null when not found')
```

Bad: `it('works correctly')` — tells you nothing when it fails.
Good: `it('sends welcome email after user registration')` — self-documenting.

---

## Mocking Strategies

### Mock only what crosses a boundary
- External APIs, databases, file systems, clocks, random — mock these
- Internal pure functions — test them directly, don't mock them

### Jest mocking patterns
```typescript
// Mock an entire module
jest.mock('../services/emailService')
import { sendEmail } from '../services/emailService'
const mockSendEmail = sendEmail as jest.MockedFunction<typeof sendEmail>

// Mock a specific return value
mockSendEmail.mockResolvedValue({ messageId: 'test-123' })

// Assert it was called correctly
expect(mockSendEmail).toHaveBeenCalledWith({
  to: 'user@example.com',
  subject: 'Welcome!'
})

// Mock implementation
mockSendEmail.mockImplementation(async ({ to }) => {
  if (to === 'bad@domain.com') throw new Error('Invalid domain')
  return { messageId: 'ok' }
})

// Reset between tests
beforeEach(() => jest.clearAllMocks())
```

### Mocking time (critical for date-dependent logic)
```typescript
// Jest fake timers
beforeEach(() => jest.useFakeTimers())
afterEach(() => jest.useRealTimers())

it('expires token after 1 hour', () => {
  const token = createToken()
  jest.advanceTimersByTime(60 * 60 * 1000 + 1) // 1hr + 1ms
  expect(isTokenExpired(token)).toBe(true)
})

// Mock Date
jest.setSystemTime(new Date('2024-06-15T12:00:00Z'))
```

### Mocking fetch / HTTP (MSW is best for integration tests)
```typescript
// Install: npm i -D msw
import { setupServer } from 'msw/node'
import { http, HttpResponse } from 'msw'

const server = setupServer(
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({ id: params.id, name: 'Alice' })
  }),
  http.post('/api/users', async ({ request }) => {
    const body = await request.json()
    return HttpResponse.json({ id: 'new-id', ...body }, { status: 201 })
  })
)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

---

## Testing Async Code

```typescript
// Always return or await promises in tests
it('fetches user data', async () => {
  const user = await getUserById('123')
  expect(user.name).toBe('Alice')
})

// Test rejection
it('throws when user not found', async () => {
  await expect(getUserById('nonexistent')).rejects.toThrow('User not found')
})

// Test callbacks (use done or wrap in Promise)
it('calls callback with result', (done) => {
  fetchUserWithCallback('123', (err, user) => {
    expect(err).toBeNull()
    expect(user.name).toBe('Alice')
    done()
  })
})
```

---

## Testing React Components

### React Testing Library (preferred — tests behavior, not implementation)
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

describe('LoginForm', () => {
  it('submits credentials on button click', async () => {
    const mockLogin = jest.fn().mockResolvedValue({ token: 'abc' })
    render(<LoginForm onLogin={mockLogin} />)

    await userEvent.type(screen.getByLabelText('Email'), 'user@example.com')
    await userEvent.type(screen.getByLabelText('Password'), 'secret123')
    await userEvent.click(screen.getByRole('button', { name: 'Log in' }))

    expect(mockLogin).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'secret123'
    })
  })

  it('shows error message on failed login', async () => {
    const mockLogin = jest.fn().mockRejectedValue(new Error('Invalid credentials'))
    render(<LoginForm onLogin={mockLogin} />)

    await userEvent.click(screen.getByRole('button', { name: 'Log in' }))

    expect(await screen.findByRole('alert')).toHaveTextContent('Invalid credentials')
  })

  it('disables submit button while loading', async () => {
    const mockLogin = jest.fn(() => new Promise(() => {})) // never resolves
    render(<LoginForm onLogin={mockLogin} />)

    await userEvent.click(screen.getByRole('button', { name: 'Log in' }))

    expect(screen.getByRole('button', { name: 'Log in' })).toBeDisabled()
  })
})
```

### Queries — priority order (most accessible to least)
1. `getByRole` — best; mirrors what screen readers see
2. `getByLabelText` — for form fields
3. `getByPlaceholderText` — fallback for inputs
4. `getByText` — for non-interactive elements
5. `getByTestId` — last resort; add `data-testid` sparingly

### Custom render wrapper (for providers)
```tsx
// test-utils.tsx
function AllProviders({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={new QueryClient()}>
      <AuthProvider>
        {children}
      </AuthProvider>
    </QueryClientProvider>
  )
}

export function renderWithProviders(ui: React.ReactElement) {
  return render(ui, { wrapper: AllProviders })
}
```

---

## Testing API Endpoints (Integration Tests)

```typescript
// Using supertest with Express
import request from 'supertest'
import { app } from '../app'
import { db } from '../db'

describe('POST /api/users', () => {
  beforeEach(async () => {
    await db.users.deleteMany() // Clean state
  })

  afterAll(async () => {
    await db.$disconnect()
  })

  it('creates a user and returns 201', async () => {
    const res = await request(app)
      .post('/api/users')
      .set('Authorization', `Bearer ${testToken}`)
      .send({ name: 'Alice', email: 'alice@example.com' })

    expect(res.status).toBe(201)
    expect(res.body).toMatchObject({ name: 'Alice', email: 'alice@example.com' })
    expect(res.body.id).toBeDefined()
  })

  it('returns 400 for missing email', async () => {
    const res = await request(app)
      .post('/api/users')
      .send({ name: 'Alice' }) // No email

    expect(res.status).toBe(400)
    expect(res.body.error).toMatch(/email/i)
  })

  it('returns 401 without auth token', async () => {
    const res = await request(app).post('/api/users').send({ name: 'Alice' })
    expect(res.status).toBe(401)
  })
})
```

---

## Testing Python (pytest)

```python
# conftest.py — shared fixtures
import pytest
from app import create_app
from db import get_db

@pytest.fixture
def app():
    app = create_app({'TESTING': True, 'DATABASE': ':memory:'})
    yield app

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.fixture
def db(app):
    with app.app_context():
        db = get_db()
        yield db
        db.execute("DELETE FROM users")
        db.commit()

# test_users.py
def test_create_user_returns_201(client):
    res = client.post('/api/users', json={'name': 'Alice', 'email': 'a@b.com'})
    assert res.status_code == 201
    assert res.json['name'] == 'Alice'

def test_create_user_missing_email_returns_400(client):
    res = client.post('/api/users', json={'name': 'Alice'})
    assert res.status_code == 400
    assert 'email' in res.json['error'].lower()

# Parametrize to test multiple inputs cleanly
@pytest.mark.parametrize('email,valid', [
    ('user@example.com', True),
    ('not-an-email', False),
    ('', False),
    ('a@b', False),
])
def test_email_validation(email, valid):
    assert is_valid_email(email) == valid

# Mocking in Python
from unittest.mock import patch, MagicMock

def test_sends_welcome_email(client):
    with patch('app.services.email.send') as mock_send:
        mock_send.return_value = {'message_id': 'test-123'}
        client.post('/api/users', json={'name': 'Alice', 'email': 'a@b.com'})
        mock_send.assert_called_once()
        assert mock_send.call_args[1]['to'] == 'a@b.com'
```

---

## Test Data Management

### Factory pattern (avoid repetitive setup)
```typescript
// factories/user.ts
let idCounter = 0

export function buildUser(overrides: Partial<User> = {}): User {
  idCounter++
  return {
    id: `user-${idCounter}`,
    name: 'Test User',
    email: `user${idCounter}@example.com`,
    role: 'standard',
    createdAt: new Date('2024-01-01'),
    ...overrides
  }
}

// In tests:
const admin = buildUser({ role: 'admin' })
const unverified = buildUser({ email: null, verified: false })
```

### Database seeding for integration tests
```typescript
// Always clean before seeding, not after
// (leaving data helps debug failures)
beforeEach(async () => {
  await db.truncate(['users', 'orders'])
  await db.users.createMany({ data: [buildUser(), buildUser()] })
})
```

---

## Coverage — What It Means and Doesn't

- **Line/statement coverage**: 80%+ is a reasonable floor; 100% is often not worth it
- **Branch coverage**: more meaningful — ensure both true/false paths are tested
- **Don't test for coverage** — test for correctness; coverage is a diagnostic tool
- Untested code paths aren't necessarily bad — untested *critical* paths are

```json
// jest.config.json — enforce minimum thresholds
{
  "coverageThreshold": {
    "global": {
      "branches": 75,
      "functions": 80,
      "lines": 80
    }
  }
}
```

---

## Snapshot Testing — Use Sparingly

```tsx
// Good: stable, intentional UI components
it('renders user card correctly', () => {
  const { container } = render(<UserCard name="Alice" role="admin" />)
  expect(container).toMatchSnapshot()
})

// Bad: large pages, components with dynamic data/dates/IDs
// Snapshots become meaningless when they break constantly
```

Only use snapshots for small, stable, pure UI components. Update them intentionally — never blindly run `jest --updateSnapshot`.

---

## Testing Checklist (before shipping)

- [ ] Happy path covered
- [ ] Error paths covered (network failure, invalid input, not found)
- [ ] Edge cases: null, empty, zero, very large, concurrent
- [ ] Auth: unauthenticated, wrong role, other user's resource (IDOR)
- [ ] All mocks reset between tests
- [ ] Tests pass in isolation (not just when run in sequence)
- [ ] No hardcoded timing (`setTimeout(done, 1000)` — use fake timers)
- [ ] Regression test added for any bug fix
