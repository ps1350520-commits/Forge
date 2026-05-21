# API Design — REST, GraphQL, Versioning & Documentation

## REST API Design Principles

### URL structure — resources, not actions
```
Resources are nouns. HTTP methods are the verbs.

✅ Good
GET    /users                    # list users
GET    /users/:id                # get one user
POST   /users                    # create user
PATCH  /users/:id                # partial update
PUT    /users/:id                # full replace
DELETE /users/:id                # delete user

GET    /users/:id/orders         # user's orders (nested resource)
GET    /orders?userId=:id        # alternative: flat with filter

❌ Bad
GET  /getUsers
POST /createUser
GET  /user/delete/:id
POST /users/:id/doActivate
```

### Nesting — max 2 levels deep
```
✅ /users/:id/orders           — one level of nesting, clear ownership
✅ /users/:id/orders/:orderId  — two levels, still readable

❌ /users/:id/orders/:orderId/items/:itemId/reviews
   — too deep, use flat routes for deeply nested resources
✅ /order-items/:itemId/reviews  — flatten with a clear resource name
```

### HTTP status codes — use them precisely

| Code | Meaning | Use for |
|---|---|---|
| 200 | OK | Successful GET, PATCH, PUT |
| 201 | Created | Successful POST — include `Location` header |
| 204 | No Content | Successful DELETE, or PATCH with no response body |
| 400 | Bad Request | Validation error, malformed request |
| 401 | Unauthorized | Not authenticated — missing/invalid token |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource, optimistic lock failure |
| 422 | Unprocessable | Valid syntax but semantic error (e.g., end date before start date) |
| 429 | Too Many Requests | Rate limited — include `Retry-After` header |
| 500 | Internal Server Error | Unexpected server error — don't expose details |

### Consistent error response shape
```typescript
// Every error response follows this structure
interface ErrorResponse {
  error: {
    code: string        // machine-readable: 'VALIDATION_ERROR', 'NOT_FOUND'
    message: string     // human-readable, safe to display
    details?: Record<string, string>  // field-level errors for forms
    requestId: string   // correlate with logs
  }
}

// Examples:
// 400 Validation error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": {
      "email": "Must be a valid email address",
      "password": "Must be at least 8 characters"
    },
    "requestId": "req_01HXZ..."
  }
}

// 404 Not found
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Order not found",
    "requestId": "req_01HXZ..."
  }
}

// 429 Rate limited
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Please retry after 60 seconds.",
    "requestId": "req_01HXZ..."
  }
}
// + Header: Retry-After: 60
```

---

## Request & Response Design

### Input validation — define your contract
```typescript
// Zod schema — serves as both validation and TypeScript type
import { z } from 'zod'

const CreateOrderSchema = z.object({
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive().max(100),
  })).min(1).max(50),
  shippingAddressId: z.string().uuid(),
  couponCode: z.string().max(50).optional(),
})

type CreateOrderInput = z.infer<typeof CreateOrderSchema>

// In route handler:
app.post('/orders', async (req, res) => {
  const result = CreateOrderSchema.safeParse(req.body)
  if (!result.success) {
    return res.status(400).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Request validation failed',
        details: formatZodErrors(result.error),
        requestId: req.id,
      }
    })
  }
  const input: CreateOrderInput = result.data
  // ...
})
```

### Pagination — always paginate list endpoints
```typescript
// Cursor-based (preferred for large datasets)
interface CursorPage<T> {
  data: T[]
  pagination: {
    nextCursor: string | null   // null = no more pages
    hasMore: boolean
  }
}

// GET /orders?cursor=eyJpZCI...&limit=20
// Response:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6IjAxSFh...",
    "hasMore": true
  }
}

// Offset-based (acceptable for small datasets / when user needs page jumping)
interface OffsetPage<T> {
  data: T[]
  pagination: {
    total: number
    page: number
    pageSize: number
    totalPages: number
  }
}
```

### Filtering, sorting, searching
```
GET /orders?status=pending&status=processing  # multi-value filter
GET /orders?createdAfter=2024-01-01&createdBefore=2024-03-31
GET /orders?sort=createdAt:desc,total:asc     # multiple sort fields
GET /products?q=running+shoes                 # full-text search
GET /products?fields=id,name,price           # sparse fieldsets
```

### Response envelope — keep it consistent
```typescript
// Option A: Plain resource (simpler)
GET /users/123 → { "id": "123", "name": "Alice", ... }

// Option B: Wrapped (allows metadata)
GET /users/123 → {
  "data": { "id": "123", "name": "Alice" },
  "meta": { "requestId": "req_01...", "version": "2024-01" }
}

// Choose one. Never mix.
```

---

## API Versioning

### URL versioning (recommended — explicit and cacheable)
```
/v1/users
/v2/users

# Version in path is easy to route, log, and test
```

### Header versioning (cleaner URLs, harder to test)
```
GET /users
Api-Version: 2024-03
```

### Versioning strategy
- **Don't version prematurely** — start with v1, version when you have a breaking change
- **Breaking change** = removing a field, changing a field type, changing behavior
- **Non-breaking** = adding optional fields, adding new endpoints, adding enum values
- **Maintain two versions max** — v1 and v2. When v3 ships, deprecate v1 with sunset date
- **Sunset header**: `Sunset: Sat, 31 Dec 2025 23:59:59 GMT`

```typescript
// Backward-compatible additions only in same version
// v1 response
{ "name": "Alice Doe" }

// Adding fields is safe (clients ignore unknown)
{ "name": "Alice Doe", "displayName": "Alice" }

// ❌ Breaking — renaming a field requires a new version
{ "fullName": "Alice Doe" }  // old clients break
```

---

## Authentication & Authorization Patterns

### JWT best practices
```typescript
// Short-lived access token + long-lived refresh token
const accessToken  = jwt.sign({ userId, role }, secret, { expiresIn: '15m' })
const refreshToken = jwt.sign({ userId, jti: uuid() }, refreshSecret, { expiresIn: '7d' })

// Store refresh tokens in DB for revocation
// httpOnly cookie for refresh token (not localStorage)
// Authorization header for access token

// Verify and extract — don't trust unverified payload
function requireAuth(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '')
  if (!token) return res.status(401).json({ error: { code: 'UNAUTHORIZED' } })

  try {
    const payload = jwt.verify(token, secret) as JwtPayload
    req.user = { id: payload.userId, role: payload.role }
    next()
  } catch {
    return res.status(401).json({ error: { code: 'TOKEN_INVALID' } })
  }
}
```

### API keys (for server-to-server)
```typescript
// Store hashed — never the raw key
const rawKey = `sk_live_${crypto.randomBytes(32).toString('hex')}`
const keyHash = await bcrypt.hash(rawKey, 10)
await db.apiKeys.create({ hash: keyHash, userId, name, createdAt: new Date() })
// Return rawKey once to user — never store it

// Verify
async function verifyApiKey(rawKey: string) {
  const keys = await db.apiKeys.findMany({ where: { userId: ... } })
  for (const key of keys) {
    if (await bcrypt.compare(rawKey, key.hash)) return key
  }
  return null
}
```

### Authorization — check ownership
```typescript
// ❌ Only checks authentication — IDOR vulnerability
app.get('/orders/:id', requireAuth, async (req, res) => {
  const order = await db.orders.findById(req.params.id)
  if (!order) return res.status(404).json(...)
  res.json(order) // Any authenticated user can see any order!
})

// ✅ Check ownership
app.get('/orders/:id', requireAuth, async (req, res) => {
  const order = await db.orders.findById(req.params.id)
  if (!order) return res.status(404).json(...)
  if (order.userId !== req.user.id && req.user.role !== 'admin') {
    return res.status(403).json({ error: { code: 'FORBIDDEN' } })
  }
  res.json(order)
})
```

---

## Rate Limiting

```typescript
import rateLimit from 'express-rate-limit'
import RedisStore from 'rate-limit-redis'

// General API rate limit
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  standardHeaders: true,     // Return rate limit info in RateLimit-* headers
  legacyHeaders: false,
  store: new RedisStore({ client: redisClient }), // Distributed — works across instances
  handler: (req, res) => res.status(429).json({
    error: { code: 'RATE_LIMITED', message: 'Too many requests', requestId: req.id }
  })
})

// Strict limit for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10, // Only 10 login attempts per 15 minutes per IP
  skipSuccessfulRequests: true,
})

app.use('/api/', apiLimiter)
app.use('/auth/login', authLimiter)
app.use('/auth/forgot-password', authLimiter)
```

---

## GraphQL Design (when REST isn't enough)

### When to use GraphQL
- ✅ Multiple clients with very different data needs (mobile vs web vs partner API)
- ✅ Highly interconnected data where over-fetching is costly
- ❌ Simple CRUD APIs — REST is less complex
- ❌ File upload heavy — REST handles this better

### Schema design principles
```graphql
# Types model the domain, not the database
type Order {
  id: ID!
  status: OrderStatus!
  items: [OrderItem!]!
  total: Money!  # Use scalar types for domain concepts
  customer: User!
  placedAt: DateTime!
}

# Enums over strings
enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

# Input types for mutations — separate from output types
input CreateOrderInput {
  items: [OrderItemInput!]!
  shippingAddressId: ID!
  couponCode: String
}

# Mutations follow a pattern: verb + noun, return union
type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderResult!
}

union CreateOrderResult = Order | ValidationError | PaymentError
```

### Production GraphQL requirements
```typescript
// 1. Depth limiting — prevent deeply nested malicious queries
import depthLimit from 'graphql-depth-limit'
const server = new ApolloServer({
  validationRules: [depthLimit(7)]
})

// 2. Query complexity limiting
import { createComplexityRule } from 'graphql-query-complexity'
const complexityRule = createComplexityRule({
  maximumComplexity: 1000,
  estimators: [fieldExtensionsEstimator(), simpleEstimator({ defaultComplexity: 1 })]
})

// 3. Disable introspection in production
const server = new ApolloServer({
  introspection: process.env.NODE_ENV !== 'production'
})

// 4. DataLoader to prevent N+1
const userLoader = new DataLoader(async (userIds: readonly string[]) => {
  const users = await db.users.findMany({ where: { id: { in: [...userIds] } } })
  return userIds.map(id => users.find(u => u.id === id) ?? null)
})
```

---

## API Documentation — OpenAPI / Swagger

### OpenAPI 3.0 spec skeleton
```yaml
openapi: 3.0.3
info:
  title: Acme API
  version: 1.0.0
  description: |
    REST API for the Acme platform.
    
    ## Authentication
    All endpoints require a Bearer token in the Authorization header.
    
    ## Rate Limiting
    100 requests per 15-minute window. Headers: RateLimit-Limit, RateLimit-Remaining.

servers:
  - url: https://api.acme.com/v1
    description: Production
  - url: https://api.staging.acme.com/v1
    description: Staging

paths:
  /orders:
    post:
      summary: Create an order
      operationId: createOrder
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderInput'
            example:
              items:
                - productId: "01HXZ123"
                  quantity: 2
              shippingAddressId: "01HXZ456"
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          $ref: '#/components/responses/ValidationError'
        '401':
          $ref: '#/components/responses/Unauthorized'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  responses:
    ValidationError:
      description: Validation failed
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
```

### Idempotency keys (for safe retries)
```typescript
// Client sends a unique key — server guarantees same result on retry
app.post('/orders', requireAuth, async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key']
  if (!idempotencyKey) {
    return res.status(400).json({ error: { code: 'IDEMPOTENCY_KEY_REQUIRED' } })
  }

  // Check cache first
  const cached = await redis.get(`idem:${idempotencyKey}`)
  if (cached) {
    return res.status(200).json(JSON.parse(cached)) // Return stored result
  }

  // Process and store result
  const order = await createOrder(req.user.id, req.body)
  const response = { data: order }
  await redis.setex(`idem:${idempotencyKey}`, 86400, JSON.stringify(response))
  res.status(201).json(response)
})
```
