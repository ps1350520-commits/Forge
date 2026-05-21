# Debugging Playbooks

## JavaScript / TypeScript (Node.js)

### Node.js Inspector (full debugger)
```bash
# Start with inspector
node --inspect server.js
node --inspect-brk server.js  # Break before first line

# Then open: chrome://inspect in Chrome
# Or use VS Code launch config:
{
  "type": "node",
  "request": "attach",
  "name": "Attach to Node",
  "port": 9229
}
```

### VS Code launch.json for common setups
```json
// Next.js
{
  "type": "node",
  "request": "launch",
  "name": "Next.js debug",
  "program": "${workspaceFolder}/node_modules/.bin/next",
  "args": ["dev"],
  "env": { "NODE_OPTIONS": "--inspect" }
}

// Jest tests
{
  "type": "node",
  "request": "launch",
  "name": "Debug Jest",
  "program": "${workspaceFolder}/node_modules/.bin/jest",
  "args": ["--runInBand", "--testNamePattern", "${input:testName}"],
  "console": "integratedTerminal"
}
```

### Useful runtime debugging utilities
```typescript
// Deep inspect an object (avoids [Object] truncation)
console.log(util.inspect(complexObject, { depth: null, colors: true }))

// Time a section of code
console.time('database-query')
await db.query(...)
console.timeEnd('database-query')  // → "database-query: 42.3ms"

// Stack trace to find who called a function
console.trace('called from here')

// Conditional breakpoint in code
if (user.id === 'problematic-id') debugger

// Memory snapshot
const { writeHeapSnapshot } = require('v8')
writeHeapSnapshot()  // Creates heapXXX.heapsnapshot for Chrome DevTools
```

### Async debugging patterns
```typescript
// Find unhandled rejections (add at app startup)
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason)
})

// Track which async operation is slow
async function tracedOperation(name: string, fn: () => Promise<any>) {
  const start = Date.now()
  try {
    const result = await fn()
    console.log(`${name} completed in ${Date.now() - start}ms`)
    return result
  } catch (err) {
    console.error(`${name} failed after ${Date.now() - start}ms`, err)
    throw err
  }
}
```

---

## React

### React DevTools
- Install browser extension: React DevTools
- **Components tab**: inspect component tree, props, state, hooks in real-time
- **Profiler tab**: record renders, find which components render too often and why

### Debugging re-renders
```tsx
// 1. Add why-did-you-render for development
import whyDidYouRender from '@welldone-software/why-did-you-render'
if (process.env.NODE_ENV === 'development') {
  whyDidYouRender(React, { trackAllPureComponents: true })
}

// 2. Mark a specific component
MyComponent.whyDidYouRender = true

// 3. Manual render tracking with useRef
function MyComponent({ data }) {
  const renderCount = useRef(0)
  renderCount.current++
  console.log(`Render #${renderCount.current}`, { data })
  // ...
}
```

### Debugging stale closures
```tsx
// Problem: useEffect captures old value
useEffect(() => {
  const interval = setInterval(() => {
    console.log(count) // Always logs initial value!
  }, 1000)
  return () => clearInterval(interval)
}, []) // ← missing dependency

// Solution 1: Add dependency
}, [count])

// Solution 2: Ref for mutable value without triggering re-render
const countRef = useRef(count)
useEffect(() => { countRef.current = count }, [count])
useEffect(() => {
  const interval = setInterval(() => {
    console.log(countRef.current) // Always fresh
  }, 1000)
  return () => clearInterval(interval)
}, [])
```

### Debugging state updates
```tsx
// Log state after every change
useEffect(() => {
  console.log('State changed:', { user, posts, loading })
}, [user, posts, loading])

// Verify immutability — find direct mutations
// Install: npm i -D @redux-devtools/extension
// Use immer for complex state updates
```

---

## Python

### pdb / breakpoint()
```python
# Drop into debugger at any point
breakpoint()  # Python 3.7+ (same as import pdb; pdb.set_trace())

# Common pdb commands:
# n    — next line (step over)
# s    — step into function
# c    — continue to next breakpoint
# l    — list surrounding code
# p x  — print value of x
# pp x — pretty-print x
# bt   — show call stack
# q    — quit

# Conditional breakpoint
if user_id == 'bad-user':
    breakpoint()
```

### Better REPL debugging with ipdb
```bash
pip install ipdb
# Then use: import ipdb; ipdb.set_trace()
# Has tab completion and syntax highlighting
```

### Logging module (better than print)
```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s %(levelname)s %(name)s: %(message)s'
)
logger = logging.getLogger(__name__)

logger.debug('Processing user %s with data: %r', user_id, data)
logger.error('Failed to process', exc_info=True)  # Includes stack trace
```

### Inspecting objects
```python
# See all attributes and methods
dir(obj)

# Get type and value
type(obj), repr(obj)

# Full object inspection
import pprint
pprint.pprint(vars(obj))

# Check if something is being modified
import copy
snapshot = copy.deepcopy(data)
# ... some code ...
assert data == snapshot, f"Data was mutated: {data}"
```

---

## Database (PostgreSQL / MySQL)

### EXPLAIN ANALYZE — find slow queries
```sql
EXPLAIN ANALYZE
SELECT u.*, p.* FROM users u
LEFT JOIN posts p ON p.user_id = u.id
WHERE u.created_at > '2024-01-01';

-- Look for:
-- Seq Scan on large tables (needs index)
-- Nested Loop with high rows estimates
-- High "actual time" values
-- "Rows Removed by Filter" (index is there but not filtering efficiently)
```

### Enable query logging (dev/staging only)
```sql
-- PostgreSQL: log all queries over 100ms
ALTER SYSTEM SET log_min_duration_statement = 100;
SELECT pg_reload_conf();

-- Check current slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Find N+1 queries
```sql
-- Enable pgBadger or check logs for repeated similar queries
-- In ORMs: enable query logging in development
-- Prisma: add to schema.prisma
datasource db {
  url = env("DATABASE_URL")
  // Add query log
}
// In code:
const prisma = new PrismaClient({ log: ['query', 'info', 'warn'] })
```

### Check for locks / deadlocks
```sql
-- PostgreSQL: current locks
SELECT pid, relation::regclass, mode, granted
FROM pg_locks
JOIN pg_stat_activity USING (pid)
WHERE relation IS NOT NULL;

-- Kill a blocking query
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'active' AND query_start < now() - interval '5 minutes';
```

---

## Network / HTTP Debugging

### curl with verbose output
```bash
# Full request and response headers
curl -v https://api.example.com/users

# POST with JSON body and auth
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Alice","email":"alice@example.com"}' \
  -v

# Time each phase of the request
curl -w "\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  -o /dev/null -s https://api.example.com/health
```

### Browser DevTools Network tab tips
- **Filter by XHR/Fetch** to see only API calls
- Click a request → **Preview** to see parsed JSON response
- **Timing** tab shows DNS, connect, TTFB, download breakdown
- Right-click a request → **Copy as cURL** to replay it in terminal
- **Preserve log**: keeps requests across page navigations

---

## Memory Leak Debugging (Node.js)

```javascript
// Capture heap snapshot
const v8 = require('v8')
const path = require('path')

function captureHeap(label) {
  const filename = path.join('/tmp', `heap-${label}-${Date.now()}.heapsnapshot`)
  v8.writeHeapSnapshot(filename)
  console.log('Heap snapshot written to:', filename)
}

// Take snapshots before and after suspected leak
captureHeap('before')
// ... do the operation ...
captureHeap('after')
// Load both in Chrome DevTools > Memory > Load profile
// Compare allocations between snapshots

// Monitor memory usage over time
setInterval(() => {
  const used = process.memoryUsage()
  console.log({
    rss: Math.round(used.rss / 1024 / 1024) + 'MB',
    heapUsed: Math.round(used.heapUsed / 1024 / 1024) + 'MB',
    heapTotal: Math.round(used.heapTotal / 1024 / 1024) + 'MB',
  })
}, 5000)
```

---

> For debugging methodology — hypothesis formation, probe discipline, narrowing loop, and post-mortem — use the **debug skill**. This file covers tooling only.
