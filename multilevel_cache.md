Here's a compact but complete implementation prompt you can hand to Gemini 2.5:
Prompt:
Implement a trade outcome resolution system with the following architecture. Use [your language/stack — specify e.g. Node.js/TypeScript, Python/FastAPI, Java/Spring].
Problem: 4 external API endpoints, each with different request shapes (key-value pair, list, plain string, etc.) and different response shapes (JSON with multiple fields, plain string, etc.). Trades arrive with params that need resolving against one of these endpoints. ~20k requests/day.
Required components:
Adapter interface — one implementation per endpoint, each providing:
canonicalize(params) → stable canonical request form
buildRequestHash(canonical) → deterministic hash string
callExternal(canonical) → raw API call
normalize(rawResponse) → { payload, resolvedAt } (payload can be any shape — string, object, etc.)
Resolution pipeline (shared across all adapters), for a given (endpointId, requestHash):
Check L1 client-side cache (short TTL, in-memory)
Check L2 shared cache (Redis) — miss falls through
Check Postgres persistence table — miss falls through
Call external API via the adapter — success writes through to Postgres (INSERT ... ON CONFLICT (endpoint_id, request_hash) DO UPDATE) then populates L2/L1
Single-flight on cache misses: concurrent requests for the same (endpointId, requestHash) must collapse into one external call (use a Redis-based lock, e.g. SET NX with short expiry + wait/poll for the result).
Per-endpoint resilience: circuit breaker, retry with exponential backoff + jitter, and timeout — independent per endpoint so one failing endpoint doesn't affect others.
Cache invalidation: a single admin/webhook endpoint that clears all of L2 (Redis) and bumps a generation counter (stored in Redis). Clients poll/check this counter and drop their entire L1 on mismatch. Postgres is never cleared by this — it's the durable fallback.
Fallback behavior when external API fails:
If Postgres has a prior resolution for that key → serve it, marked stale: true in the response envelope, and cache it briefly (short TTL) in L2 to avoid re-hammering the API
If no prior resolution exists anywhere (never-seen combo + API down) → this is a hard miss: [pick one — reject the trade immediately / queue it for background retry]. Implement as: [your choice]
Schema:
CREATE TABLE resolution_cache (
    endpoint_id TEXT NOT NULL,
    request_hash TEXT NOT NULL,
    raw_request JSONB NOT NULL,
    response_payload JSONB,
    resolved_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (endpoint_id, request_hash)
);
Observability: log/emit metrics per endpoint for cache hit/miss (L1/L2/DB/API), circuit breaker state changes, stale-serves, and hard-misses.
Provide the code structured as: adapters, resolution pipeline/service, Redis cache client wrapper, Postgres repository, circuit breaker wrapper, and a simple API layer to trigger a resolution request and the invalidation webhook.
Fill in the bracketed choices (stack, hard-miss policy) before sending — Gemini will do better with those decided rather than left open.