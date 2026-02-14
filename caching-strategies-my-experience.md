# Caching Strategies 

Caching is often introduced as a performance optimization.
In real systems, it is frequently a **correctness, scale, and operational control mechanism**.

This note documents a caching strategy I used in a production system where:
- traffic was high (~5000 TPS)
- correctness was mandatory
- the primary risk was **database collapse under spikes**

This is **not** a latency-driven cache.
It is a **correctness-first cache used as a DB shock absorber**.

---

## Case 1: Correctness-first cache for parent → child account mapping

### Problem context
An API accepted a **parent account ID** and performed **opt-in / opt-out**
operations across all associated **child accounts**.

Key constraints:
- Parent → child mapping stored in DB
- Same parent accounts were repeatedly queried
- API expected sustained high throughput (~5000 TPS)
- DB overload during spikes was the main failure risk

Correctness was **business-critical**:
- Partial or incorrect propagation would break message delivery
- “Eventually correct” was not acceptable for execution paths

---

## What was cached
- **Parent account ID → list of child account IDs**

Source of truth:
- **Database**

The cache was **never treated as authoritative**.
Its sole purpose was to reduce pressure on the DB.

---

## Cache placement & topology
- Cache located at the **API layer**
- **Shared Redis cache**
- Single region

Cache key: parentAccountId → childAccountId[]


This design limited cardinality to the number of parent accounts
and avoided per-request or per-transaction keys.

---

## Access pattern
- **Cache-aside (lazy loading)**
  - On hit: read mapping from Redis and proceed
  - On miss: fetch from DB and populate Redis

### Negative caching
To protect the DB from repeated invalid requests:
- If an **incorrect parent account ID** was requested
  more than **10 times within 5 minutes**,
  a **negative cache entry** was written to Redis.

This penalized bad input without impacting valid traffic.

---

## Freshness & update strategy
- Redis TTL: **1 hour**
- A background thread ran **hourly** to:
  - Read parent–child mappings from DB
  - Refresh Redis entries in bulk

There was **no event-driven invalidation**.
Freshness was managed via:
- bounded TTL
- scheduled refresh

This was a deliberate choice to keep the system simple and predictable
under high load.

---

## Explicit operational dependency
Parent–child mappings were modified by **support teams**.

To handle correctness-sensitive updates:
- A **dedicated cache refresh API** was exposed
- Support teams followed an **operational handbook**
- After adding a new account or changing mappings,
  support explicitly triggered cache refresh

Cache correctness therefore depended on:
- TTL + refresh job
- **human-in-the-loop operational action**

This dependency was intentional and documented.

---

## Failure behavior
- Cache miss or cold cache → fallback to DB
- Observed risk: **DB CPU spike** during heavy miss scenarios

### Safeguard
- **Rate limiting on the DB fallback path**
  - Prevented cache miss storms from overwhelming the DB
  - Ensured cache failures degraded gracefully instead of cascading

No Redis outages or cache stampede incidents were observed,
but the design assumed they were possible.

---

## Observability signals
Primary signals tracked:
- Redis **CPU**
- Redis **memory**
- Redis **connection count**

Cache hit rate was not treated as a success metric.
Cache stability mattered more than raw speed.

---

## Trade-offs accepted (explicit)
- Bounded staleness (up to 1 hour) in exchange for DB protection
- Operational dependency on support teams
- Additional complexity (negative caching, refresh job, fallback rate limits)

These were **explicitly accepted** to meet correctness and scale requirements.

---

## Review questions this case informs
- When should caching exist purely to protect a database?
- How much operational dependency is acceptable for correctness?
- Should rate limiting live on the API edge or the cache miss path?
- When is TTL + manual refresh preferable to event-driven invalidation?

---

## My takeaway
Not all caches are about speed.

Some caches exist to:
- absorb load
- penalize bad input
- protect critical data stores

If correctness is mandatory and traffic is high,
a cache can be a **defensive system boundary**, not an optimization.

Treat such caches as part of your **reliability strategy**, not your performance stack.
