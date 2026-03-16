# Add Rate Limiting to API

Add rate limiting to the public REST API to prevent abuse.

## Plan Review Summary

**Reviewed**: 2026-03-17
**Review depth**: standard
**Gaps found**: 4
**Edge cases identified**: 3
**Simulation issues**: 1
**UI challenges**: skipped
**Questions for user**: 2

### Key Findings
- No storage backend specified for the rate limit counters (in-memory won't survive restarts or work across multiple instances)
- Health check and webhook endpoints are rate limited unnecessarily
- Missing `Retry-After` header (required by RFC 6585)
- No handling for proxied requests — `X-Forwarded-For` is not being used, so all users behind a load balancer share one limit

## Proposed Changes

### API Middleware

#### [NEW] src/middleware/rateLimiter.ts

Create a rate limiter middleware using a sliding window algorithm:

- 100 requests per minute per IP
- Return `429 Too Many Requests` when exceeded
- Add `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers

> [!IMPORTANT]
> **Added by review**: Add `Retry-After` header to 429 responses (required by RFC 6585). Value should be the number of seconds until the window resets.

> [!IMPORTANT]
> **Added by review**: Use `X-Forwarded-For` header (first entry) when behind a reverse proxy, falling back to `req.ip`. Add a `trustProxy` config option to control this behavior, defaulting to `false`.

#### [MODIFY] src/server.ts

Register the rate limiter middleware before all route handlers:

```typescript
app.use(rateLimiter());
```

> [!IMPORTANT]
> **Added by review**: Exclude health check (`/health`, `/ready`) and webhook (`/webhooks/*`) routes from rate limiting. These are called by infrastructure and payment providers, not end users.

---

### Storage Backend

> [!IMPORTANT]
> **Added by review**: The original plan doesn't specify where rate limit counters are stored. In-memory storage will reset on every deploy and won't work with multiple server instances.

#### [NEW] src/middleware/rateLimitStore.ts

Create a `RateLimitStore` interface with two implementations:
- `MemoryStore` — for local development and single-instance deploys
- `RedisStore` — for production, using the existing Redis connection from `src/config/redis.ts`

The store is injected into the rate limiter middleware via config:

```typescript
app.use(rateLimiter({
  store: new RedisStore(redisClient),
  windowMs: 60_000,
  max: 100,
}));
```

#### [MODIFY] src/config/redis.ts

No code changes needed — the existing `redisClient` export is sufficient. Just import it in `server.ts`.

---

### Configuration

> [!IMPORTANT]
> **Added by review**: Rate limit values should be configurable, not hardcoded.

#### [MODIFY] src/config/env.ts

Add environment variables:

```
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=100
RATE_LIMIT_TRUST_PROXY=false
RATE_LIMIT_STORE=memory   # "memory" or "redis"
```

---

## Edge Cases Identified

1. **Multiple instances**: If the app runs behind a load balancer with 3 instances using in-memory stores, a user gets 300 requests/minute instead of 100. → Fixed by adding Redis store option.

2. **Clock skew**: If using Redis across regions, slight clock differences could cause windows to misalign. → Document that all instances should use the same Redis instance.

3. **IPv6 vs IPv4**: The same user may appear as different IPs depending on network. → Accept this as a known limitation; document it.

## Verification Plan

### Automated Tests
- Unit test the sliding window counter logic
- Integration test that the 101st request returns 429

> [!IMPORTANT]
> **Added by review**: Additional test scenarios needed.

- Test that `Retry-After` header is present and accurate on 429 responses
- Test that health check routes are not rate limited
- Test that `X-Forwarded-For` is used when `trustProxy` is enabled
- Test Redis store failover — if Redis is down, rate limiter should fail open (allow requests) rather than fail closed (block everything)
- Load test: verify counter accuracy under concurrent requests

## Review Questions

1. **Fail-open or fail-closed?** If the Redis store becomes unavailable, should the rate limiter allow all requests through (fail-open, better availability) or block all requests (fail-closed, better security)? I've assumed fail-open above — confirm?

2. **Authenticated vs. unauthenticated rates?** Should logged-in users get a higher rate limit than anonymous users? The current plan treats all requests equally. If different tiers are needed, the implementation will need to key on user ID in addition to IP.
