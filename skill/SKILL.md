---
name: backend-patterns
description: Backend architecture patterns, API design, database optimization, and server-side best practices for Node.js, Express, and Next.js API routes.
---

# Backend Development Patterns

Design and implement maintainable backend systems for Node.js, Express, and Next.js. Preserve the project's existing framework, conventions, and dependency choices.

## When to Activate

Use this skill when:

- Designing REST or GraphQL APIs
- Separating handlers, services, and data access
- Optimizing queries, indexes, pagination, or connection usage
- Preventing N+1 queries
- Adding caching or background jobs
- Implementing authentication, authorization, validation, or rate limiting
- Standardizing errors, retries, idempotency, or observability
- Reviewing backend code for correctness, security, or scalability

## Workflow

1. Inspect the existing routes, data model, error format, authentication, and tests.
2. Identify invariants, trust boundaries, expected load, and failure modes.
3. Choose the smallest architecture that cleanly separates transport, business logic, and persistence.
4. Validate all untrusted input at the boundary.
5. Implement authorization inside the request path, not only in the client.
6. Make related writes atomic and retry-safe.
7. Add focused tests for success, validation, authorization, conflicts, and dependency failures.
8. Check query count, selected columns, pagination behavior, cache invalidation, and sensitive logging.

Do not introduce new layers, dependencies, infrastructure, or external services unless the task requires them.

## API Design

### Resource-Oriented Routes

```text
GET    /api/markets                 List markets
GET    /api/markets/:id             Get one market
POST   /api/markets                 Create a market
PUT    /api/markets/:id             Replace a market
PATCH  /api/markets/:id             Update selected fields
DELETE /api/markets/:id             Delete a market

GET /api/markets?status=active&sort=-volume&limit=20&cursor=abc123
```

Use nouns for resources and HTTP methods for actions. If an operation does not map cleanly to CRUD, model it as a subresource or explicit action:

```text
POST /api/markets/:id/archive
POST /api/markets/:id/orders
```

Apply these rules:

- Return `200` for successful reads and updates.
- Return `201` with a `Location` header for creation.
- Return `204` only when the response has no body.
- Return `400` for malformed input, `401` for missing or invalid authentication, `403` for denied access, `404` for absent resources, `409` for state conflicts, and `429` for rate limits.
- Keep error envelopes consistent.
- Set maximum page sizes. Prefer cursor pagination when rows can be inserted during traversal.
- Define stable sorting with a unique tie-breaker, such as `created_at DESC, id DESC`.
- Treat `PUT` as full replacement and `PATCH` as partial modification.
- Use idempotency keys for create operations that clients may safely retry.
- Version intentionally. Prefer backward-compatible additions before introducing a new API version.

For GraphQL, apply the same boundary rules and prevent N+1 queries with batching or data loaders.

## Layer Responsibilities

Use only the layers justified by the application:

- **Handler or controller:** Parse transport input, invoke the service, and map results to HTTP.
- **Service:** Enforce business rules, authorization decisions, and transaction boundaries.
- **Repository:** Encapsulate database queries and persistence-specific behavior.
- **Job:** Perform deferred work with explicit retry and idempotency rules.

Do not put business rules in middleware or repositories merely to avoid creating a service.

### Repository Pattern

```typescript
interface MarketRepository {
  findById(id: string): Promise<Market | null>
  list(input: ListMarketsInput): Promise<Market[]>
  create(input: CreateMarketInput): Promise<Market>
  update(id: string, input: UpdateMarketInput): Promise<Market | null>
}

class SqlMarketRepository implements MarketRepository {
  constructor(private db: DatabaseClient) {}

  async findById(id: string): Promise<Market | null> {
    return this.db.market.findUnique({
      where: { id },
      select: {
        id: true,
        name: true,
        status: true,
        volume: true,
      },
    })
  }

  async list(input: ListMarketsInput): Promise<Market[]> {
    return this.db.market.findMany({
      where: input.status ? { status: input.status } : undefined,
      orderBy: [{ volume: 'desc' }, { id: 'desc' }],
      take: Math.min(input.limit, 100),
    })
  }

  // Implement create and update using the same domain mapping.
}
```

Keep database models from leaking across the application when persistence fields differ from domain fields.

### Service Pattern

```typescript
class MarketService {
  constructor(private markets: MarketRepository) {}

  async renameMarket(
    actor: AuthenticatedUser,
    marketId: string,
    name: string
  ): Promise<Market> {
    const market = await this.markets.findById(marketId)

    if (!market) {
      throw new ApiError(404, 'MARKET_NOT_FOUND', 'Market not found')
    }

    if (!actor.permissions.includes('markets:update')) {
      throw new ApiError(403, 'FORBIDDEN', 'You cannot update this market')
    }

    const updated = await this.markets.update(marketId, { name })

    if (!updated) {
      throw new ApiError(409, 'MARKET_CHANGED', 'Market changed during update')
    }

    return updated
  }
}
```

Do not authorize from caller-provided ownership fields. Load trusted ownership or permission data first.

## Validation and Middleware

Validate path parameters, query parameters, headers, and bodies before invoking business logic. Reject unknown fields when silent acceptance could hide client mistakes or mass-assignment vulnerabilities.

```typescript
const updateMarketSchema = z.object({
  name: z.string().trim().min(1).max(120),
  status: z.enum(['active', 'closed']).optional(),
}).strict()

export function requireAuth(
  handler: (req: AuthenticatedRequest, res: Response) => Promise<void>
) {
  return async (req: Request, res: Response) => {
    const match = req.headers.authorization?.match(/^Bearer (.+)$/)

    if (!match) {
      res.status(401).json({
        error: { code: 'UNAUTHORIZED', message: 'Authentication required' },
      })
      return
    }

    try {
      const user = await verifyToken(match[1])
      await handler(Object.assign(req, { user }), res)
    } catch {
      res.status(401).json({
        error: { code: 'INVALID_TOKEN', message: 'Invalid credentials' },
      })
    }
  }
}
```

Authentication establishes identity. Authorization must separately verify what that identity may do.

Middleware must either send a response or continue the chain exactly once. Avoid catching application errors in authentication middleware unless they are authentication failures.

## Database Patterns

### Query Efficiently

```typescript
const markets = await db.market.findMany({
  where: { status: 'active' },
  select: {
    id: true,
    name: true,
    status: true,
    volume: true,
  },
  orderBy: [{ volume: 'desc' }, { id: 'desc' }],
  take: 20,
})
```

- Select only required columns.
- Bound every user-controlled limit.
- Add indexes that match frequent filters, joins, and ordering.
- Verify indexes with the database query plan.
- Avoid offset pagination for deep or frequently changing result sets.
- Do not add indexes blindly. Each index increases storage and write cost.
- Enforce invariants with database constraints as well as application validation.
- Use connection pooling and avoid creating a client per request.
- Never interpolate untrusted values into SQL. Use parameters or the query builder.

### Prevent N+1 Queries

```typescript
const markets = await getMarkets()
const creatorIds = [...new Set(markets.map(market => market.creatorId))]
const creators = await getUsersByIds(creatorIds)
const creatorById = new Map(creators.map(creator => [creator.id, creator]))

const result = markets.map(market => ({
  ...market,
  creator: creatorById.get(market.creatorId) ?? null,
}))
```

Use joins, relation includes, or batched lookups. Preserve result ordering explicitly because batched queries may return rows in a different order.

### Transactions

Use a transaction when multiple writes must succeed or fail together:

```typescript
async function createMarketWithPosition(
  input: CreateMarketWithPositionInput
) {
  return db.$transaction(async tx => {
    const market = await tx.market.create({
      data: input.market,
    })

    const position = await tx.position.create({
      data: {
        ...input.position,
        marketId: market.id,
      },
    })

    return { market, position }
  })
}
```

Keep transactions short. Do not perform slow network requests inside them. Handle unique constraint and serialization conflicts explicitly. Use optimistic concurrency, such as a version column, when concurrent updates could overwrite each other.

## Caching

Prefer cache-aside for read-heavy data:

```typescript
async function getMarket(id: string): Promise<Market | null> {
  const key = `market:v1:${id}`
  const cached = await redis.get(key)

  if (cached !== null) {
    return JSON.parse(cached) as Market
  }

  const market = await marketRepository.findById(id)

  if (market) {
    await redis.set(key, JSON.stringify(market), { EX: 300 })
  }

  return market
}

async function updateMarket(
  id: string,
  input: UpdateMarketInput
): Promise<Market> {
  const market = await marketRepository.update(id, input)

  if (!market) {
    throw new ApiError(404, 'MARKET_NOT_FOUND', 'Market not found')
  }

  await redis.del(`market:v1:${id}`)
  return market
}
```

Define before adding a cache:

- The cache key and version
- TTL and acceptable staleness
- Invalidation for every write path
- Behavior when the cache is unavailable
- Whether missing values should be cached briefly
- Serialization and schema compatibility

A cache failure should usually degrade performance, not break correctness. Prevent cache stampedes for expensive hot keys with request coalescing, locking, or randomized TTLs.

Do not cache private responses in shared caches unless the key includes the relevant identity and authorization scope.

## Errors and Logging

Use stable machine-readable codes and safe client messages:

```typescript
class ApiError extends Error {
  constructor(
    readonly status: number,
    readonly code: string,
    message: string,
    readonly details?: unknown
  ) {
    super(message)
    this.name = 'ApiError'
  }
}

function errorResponse(error: unknown, requestId: string): Response {
  if (error instanceof z.ZodError) {
    return Response.json(
      {
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Request validation failed',
          details: error.flatten(),
          requestId,
        },
      },
      { status: 400 }
    )
  }

  if (error instanceof ApiError) {
    return Response.json(
      {
        error: {
          code: error.code,
          message: error.message,
          details: error.details,
          requestId,
        },
      },
      { status: error.status }
    )
  }

  console.error('Unexpected request failure', { requestId, error })

  return Response.json(
    {
      error: {
        code: 'INTERNAL_ERROR',
        message: 'Internal server error',
        requestId,
      },
    },
    { status: 500 }
  )
}
```

Never expose stack traces, SQL messages, tokens, cookies, passwords, or internal identifiers to clients. Redact secrets from logs. Include a request or trace ID so client failures can be correlated with server logs.

## Retries, Timeouts, and Jobs

Retry only transient failures and only when the operation is idempotent:

```typescript
async function retry<T>(
  operation: () => Promise<T>,
  options: {
    attempts?: number
    baseDelayMs?: number
    shouldRetry: (error: unknown) => boolean
  }
): Promise<T> {
  const attempts = options.attempts ?? 3
  const baseDelayMs = options.baseDelayMs ?? 100

  if (attempts < 1) {
    throw new Error('attempts must be at least 1')
  }

  let lastError: unknown

  for (let attempt = 0; attempt < attempts; attempt += 1) {
    try {
      return await operation()
    } catch (error) {
      lastError = error

      if (
        attempt === attempts - 1 ||
        !options.shouldRetry(error)
      ) {
        throw error
      }

      const exponentialDelay = baseDelayMs * 2 ** attempt
      const jitter = Math.random() * exponentialDelay
      await new Promise(resolve => setTimeout(resolve, jitter))
    }
  }

  throw lastError
}
```

Set timeouts for database and dependency calls. Do not retry validation failures, authorization failures, most `4xx` responses, or non-idempotent writes without an idempotency mechanism. Respect server retry hints when available.

Background jobs should:

- Carry a stable job or idempotency key
- Be safe to execute more than once
- Record attempts and terminal failure
- Use bounded exponential backoff with jitter
- Move exhausted jobs to a dead-letter or review state
- Avoid acknowledging completion before durable work finishes
- Support graceful shutdown without abandoning in-progress work

## Concrete Usage Example

Implementing `PATCH /api/markets/:id` in an Express application:

```typescript
const paramsSchema = z.object({
  id: z.string().uuid(),
})

const bodySchema = z.object({
  name: z.string().trim().min(1).max(120),
}).strict()

app.patch(
  '/api/markets/:id',
  requireAuth(async (req, res) => {
    const params = paramsSchema.parse(req.params)
    const body = bodySchema.parse(req.body)

    const market = await marketService.renameMarket(
      req.user,
      params.id,
      body.name
    )

    res.status(200).json({ data: market })
  })
)
```

For this endpoint, verify:

- Invalid UUIDs and bodies return `400`.
- Missing authentication returns `401`.
- Insufficient permission returns `403`.
- Missing markets return `404`.
- Concurrent modification returns `409` when applicable.
- Unknown body fields are rejected.
- The repository selects only fields needed by the response.
- Any cached market is invalidated after a successful update.
- Tests cover each outcome and confirm the service is not called after boundary validation fails.

## Review Checklist

Before considering backend work complete, confirm:

- Inputs are validated and outputs use a consistent contract.
- Authentication and authorization are separate and server-enforced.
- Queries are bounded, parameterized, and free of N+1 behavior.
- Data invariants have database constraints where practical.
- Related writes are atomic.
- Retried writes are idempotent.
- Cache invalidation covers every mutation path.
- Errors do not leak sensitive implementation details.
- Logs contain correlation context but no secrets.
- Timeouts, cancellation, and graceful shutdown are considered.
- Tests cover failure paths, not only successful requests.