# Common Design Patterns — Quick Reference

## Creational

### Factory
Use when: you need to create objects without specifying the exact class.
```typescript
interface Logger { log(msg: string): void }
class ConsoleLogger implements Logger { log = (msg: string) => console.log(msg) }
class FileLogger implements Logger { log = (msg: string) => fs.appendFileSync('app.log', msg) }

function createLogger(env: string): Logger {
  return env === 'production' ? new FileLogger() : new ConsoleLogger()
}
```

### Singleton (use sparingly)
Use when: exactly one instance must exist (DB connection pool, config).
```typescript
class Database {
  private static instance: Database
  private constructor(private url: string) {}
  static getInstance(url: string) {
    if (!Database.instance) Database.instance = new Database(url)
    return Database.instance
  }
}
```

## Structural

### Repository
Use when: abstracting data access from business logic.
```typescript
interface UserRepository {
  findById(id: string): Promise<User | null>
  save(user: User): Promise<void>
}

class PostgresUserRepository implements UserRepository {
  constructor(private db: Pool) {}
  async findById(id: string) {
    const { rows } = await this.db.query('SELECT * FROM users WHERE id=$1', [id])
    return rows[0] ?? null
  }
  async save(user: User) {
    await this.db.query(
      'INSERT INTO users(id,name,email) VALUES($1,$2,$3) ON CONFLICT(id) DO UPDATE SET name=$2,email=$3',
      [user.id, user.name, user.email]
    )
  }
}
```

### Adapter
Use when: making incompatible interfaces work together.
```typescript
// Third-party payment API has a different interface
class StripePayment {
  chargeCard(amount: number, token: string) { /* stripe logic */ }
}

// Our app expects this interface
interface PaymentProvider {
  charge(amountCents: number, paymentToken: string): Promise<void>
}

class StripeAdapter implements PaymentProvider {
  constructor(private stripe: StripePayment) {}
  async charge(amountCents: number, paymentToken: string) {
    this.stripe.chargeCard(amountCents / 100, paymentToken)
  }
}
```

## Behavioral

### Observer / Event Emitter
Use when: decoupling components that need to react to events.
```typescript
type Handler<T> = (data: T) => void

class EventBus {
  private listeners = new Map<string, Handler<any>[]>()

  on<T>(event: string, handler: Handler<T>) {
    const handlers = this.listeners.get(event) ?? []
    this.listeners.set(event, [...handlers, handler])
  }

  emit<T>(event: string, data: T) {
    this.listeners.get(event)?.forEach(h => h(data))
  }
}
```

### Strategy
Use when: you want to swap algorithms at runtime.
```typescript
interface SortStrategy<T> {
  sort(items: T[]): T[]
}

class QuickSort<T> implements SortStrategy<T> {
  sort(items: T[]) { return [...items].sort() }
}

class Sorter<T> {
  constructor(private strategy: SortStrategy<T>) {}
  setStrategy(s: SortStrategy<T>) { this.strategy = s }
  sort(items: T[]) { return this.strategy.sort(items) }
}
```

### Middleware Chain (used in Express, Koa, etc.)
Use when: building pipelines of processing steps.
```typescript
type Middleware = (ctx: Context, next: () => Promise<void>) => Promise<void>

async function compose(middlewares: Middleware[], ctx: Context) {
  let index = -1
  async function dispatch(i: number): Promise<void> {
    if (i <= index) throw new Error('next() called multiple times')
    index = i
    const fn = middlewares[i]
    if (!fn) return
    await fn(ctx, () => dispatch(i + 1))
  }
  return dispatch(0)
}
```

## Frontend Patterns

### Container / Presenter (React)
```tsx
// Presenter: pure UI, no data fetching
function UserCard({ name, email, onEdit }: UserCardProps) {
  return (
    <div>
      <h2>{name}</h2>
      <p>{email}</p>
      <button onClick={onEdit}>Edit</button>
    </div>
  )
}

// Container: handles data and state
function UserCardContainer({ userId }: { userId: string }) {
  const { data, isLoading } = useQuery(['user', userId], () => fetchUser(userId))
  const [editing, setEditing] = useState(false)
  if (isLoading) return <Spinner />
  return <UserCard {...data} onEdit={() => setEditing(true)} />
}
```

### Custom Hook for shared logic
```tsx
function useDebounce<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delayMs)
    return () => clearTimeout(timer)
  }, [value, delayMs])
  return debounced
}
```
