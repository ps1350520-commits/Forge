# Performance Tips by Domain

## JavaScript / TypeScript

### Avoid N+1 patterns
```typescript
// ❌ N+1: 1 query for users + N queries for posts
const users = await db.users.findMany()
for (const user of users) {
  user.posts = await db.posts.findMany({ where: { userId: user.id } })
}

// ✅ Single query with JOIN or include
const users = await db.users.findMany({ include: { posts: true } })
```

### Use appropriate data structures
```typescript
// ❌ O(n) lookup in array
const isAllowed = allowedIds.includes(userId)

// ✅ O(1) lookup with Set
const allowedSet = new Set(allowedIds)
const isAllowed = allowedSet.has(userId)
```

### Debounce and throttle
```typescript
// For search inputs, resize handlers, etc.
const debouncedSearch = debounce(handleSearch, 300)
const throttledScroll = throttle(handleScroll, 100)
```

### Avoid blocking the event loop (Node.js)
```typescript
// ❌ Synchronous file read on each request
const data = fs.readFileSync('config.json')

// ✅ Async, or read once at startup
const data = await fs.promises.readFile('config.json', 'utf8')
```

## React

### Prevent unnecessary re-renders
```tsx
// Memoize expensive components
const HeavyList = memo(({ items }: { items: Item[] }) => (
  <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>
))

// Stable callbacks
const handleClick = useCallback(() => {
  doSomething(id)
}, [id])

// Expensive calculations
const sorted = useMemo(() => items.sort(byDate), [items])
```

### Code splitting
```tsx
// Lazy load routes and heavy components
const Dashboard = lazy(() => import('./Dashboard'))
const HeavyChart = lazy(() => import('./HeavyChart'))

// Wrap in Suspense
<Suspense fallback={<Spinner />}>
  <Dashboard />
</Suspense>
```

### Virtualize long lists
```tsx
// For lists with 100+ items, use react-window or react-virtual
import { FixedSizeList } from 'react-window'

<FixedSizeList height={600} itemCount={items.length} itemSize={50} width="100%">
  {({ index, style }) => <Row style={style} item={items[index]} />}
</FixedSizeList>
```

## Database

### Indexing strategy
```sql
-- Index columns used in WHERE, JOIN, ORDER BY, GROUP BY
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Composite index: column order matters (most selective first, or match query pattern)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Check if indexes are being used
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
```

### Select only what you need
```typescript
// ❌ Fetches all columns including large blobs
const users = await db.users.findMany()

// ✅ Only fetch needed fields
const users = await db.users.findMany({
  select: { id: true, name: true, email: true }
})
```

### Pagination
```typescript
// Offset pagination (simple but slow on large offsets)
const users = await db.users.findMany({ skip: page * limit, take: limit })

// Cursor pagination (better for large datasets)
const users = await db.users.findMany({
  take: limit,
  skip: cursor ? 1 : 0,
  cursor: cursor ? { id: cursor } : undefined,
  orderBy: { id: 'asc' }
})
```

## Caching

### Cache patterns
```typescript
// Cache-aside (most common)
async function getUser(id: string): Promise<User> {
  const cached = await redis.get(`user:${id}`)
  if (cached) return JSON.parse(cached)
  
  const user = await db.users.findUnique({ where: { id } })
  await redis.setex(`user:${id}`, 300, JSON.stringify(user)) // 5 min TTL
  return user
}

// Invalidate on update
async function updateUser(id: string, data: Partial<User>) {
  const user = await db.users.update({ where: { id }, data })
  await redis.del(`user:${id}`)
  return user
}
```

## Network / API

### Batch requests
```typescript
// ❌ Waterfall: each request waits for previous
const user = await fetchUser(id)
const posts = await fetchPosts(id)
const friends = await fetchFriends(id)

// ✅ Parallel
const [user, posts, friends] = await Promise.all([
  fetchUser(id),
  fetchPosts(id),
  fetchFriends(id)
])
```

### Response compression
```typescript
// Express
import compression from 'compression'
app.use(compression())
```

## Frontend Bundle

- Tree-shake: import named exports, not entire libraries (`import { debounce } from 'lodash-es'`)
- Analyze bundle: `npx vite-bundle-visualizer` or `webpack-bundle-analyzer`
- Images: use WebP/AVIF, lazy load with `loading="lazy"`, appropriate sizes
- Fonts: `font-display: swap`, subset fonts, preload critical fonts
