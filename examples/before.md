# Add Rate Limiting to API

Add rate limiting to the public REST API to prevent abuse.

## Proposed Changes

### API Middleware

#### [NEW] src/middleware/rateLimiter.ts

Create a rate limiter middleware using a sliding window algorithm:

- 100 requests per minute per IP
- Return `429 Too Many Requests` when exceeded
- Add `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers

#### [MODIFY] src/server.ts

Register the rate limiter middleware before all route handlers:

```typescript
app.use(rateLimiter());
```

## Verification Plan

### Automated Tests
- Unit test the sliding window counter logic
- Integration test that the 101st request returns 429
