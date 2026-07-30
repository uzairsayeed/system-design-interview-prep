# Detailed TinyURL Architecture

## 1. APIs

```http
POST   /v1/links
GET    /v1/links/{link_id}
GET    /v1/links?cursor=...
PATCH  /v1/links/{link_id}
POST   /v1/links/{link_id}/disable
POST   /v1/links/{link_id}/enable
DELETE /v1/links/{link_id}
GET    /v1/links/{link_id}/analytics
GET    /{short_code}
```

Create request:

```json
{
  "destination_url": "https://example.com/a/long/path",
  "custom_alias": "my-link",
  "expires_at": "2027-01-01T00:00:00Z",
  "redirect_type": 302
}
```

## 2. Service decomposition

### Management Service

Handles creation, custom aliases, updates, disable/enable, deletion, listing, details, quotas, and analytics queries.

### Redirect Service

Handles the latency-sensitive `GET /{short_code}` path, cache lookup, lifecycle validation, redirect response, and asynchronous click-event publication.

## 3. Short-code generation

Use an 8-character random Base62 code:

```text
a-z A-Z 0-9
```

Namespace size:

```text
62^8 ≈ 218 trillion
```

Generation flow:

```text
Generate random code
→ INSERT using UNIQUE(short_code)
→ retry internally on collision
```

Custom aliases use the same unique namespace. A custom-alias collision returns `409 Conflict`.

## 4. Creation transaction

In one PostgreSQL transaction:

1. Enforce user or anonymous quota.
2. Insert the short link.
3. Persist idempotency information.
4. Insert a lifecycle outbox event.
5. Commit.

Return `201 Created` only after commit. Cache warming is best effort.

## 5. Core data model

```text
short_links
-----------
link_id
owner_type
owner_user_id
short_code
destination_url
destination_url_hash
status
redirect_type
custom_alias
idempotency_key
request_hash
created_at
updated_at
expires_at
disabled_at
deleted_at
version
```

Constraints:

```sql
UNIQUE(short_code)
UNIQUE(owner_user_id, idempotency_key)
```

## 6. Redirect path

```text
CDN
→ application-local L1
→ distributed Redis
→ redirect datastore
```

Cached entries include:

```text
destination_url
redirect_type
status
expires_at
version
```

Redirect flow:

1. Resolve short code from cache or datastore.
2. Verify `status = ACTIVE`.
3. Verify expiry is absent or in the future.
4. Return `301` or `302` using the `Location` header.
5. Emit click analytics asynchronously.

The redirect does not change the link state.

## 7. Analytics

Redirect instances publish click events to a high-throughput stream.

```json
{
  "event_id": "evt-123",
  "link_id": "link-456",
  "short_code": "aZ91Kx2P",
  "occurred_at": "...",
  "country_code": "IN",
  "referrer_domain": "example.com",
  "device_type": "mobile",
  "visitor_fingerprint": "..."
}
```

Analytics failure must not fail redirects. CDN-served clicks are captured using CDN logs or edge events.

## 8. Cache invalidation

Management updates commit:

```text
BEGIN
- update link row
- insert lifecycle outbox event
COMMIT
```

Lifecycle events include:

```text
LINK_CREATED
LINK_UPDATED
LINK_DISABLED
LINK_ENABLED
LINK_DELETED
LINK_EXPIRY_CHANGED
```

Dedicated invalidation consumers:

- delete or refresh Redis entries
- clear application L1 entries
- purge CDN entries
- apply events only when event version is not older than cache version

Bounded TTLs protect against missed invalidations.

## 9. Thundering herd protection

Cache-aside alone does not prevent many simultaneous misses from reaching the datastore.

Use request coalescing:

```text
many cache misses
→ one loader queries datastore
→ others wait briefly
→ loader populates cache
```

Implementation options:

- in-process single-flight
- Redis `SET NX` loading gate
- CDN origin request collapsing

## 10. Viral-link handling

For a link receiving 200,000 requests/sec:

- CDN edge caching absorbs most traffic
- local L1 cache protects Redis
- Redis read replicas spread reads
- exceptional hot keys may be replicated across multiple cache keys
- heavy-hitter detection triggers prewarming or replication
- analytics producers batch events

## 11. Failure handling

### Redis unavailable

- continue using CDN and L1
- open a Redis circuit breaker
- fall back to datastore
- use request coalescing and backpressure

### Datastore unavailable

- serve valid cached mappings
- optionally allow bounded stale-if-error
- return `503` on cache miss
- never convert an unknown outage result into `404`

### Primary unavailable

Existing cached redirects continue. Creation, update, disable, deletion, and alias claims return `503` until failover.

### Analytics unavailable

Continue redirects. Buffer within strict limits and track dropped events.

## 12. Multi-region

Use one authoritative write path for:

- generated-code creation
- custom-alias claims
- destination changes
- disable and deletion

Replicate redirect mappings asynchronously to:

- regional read stores
- regional Redis clusters
- application L1 caches
- CDN edges

Existing redirects remain globally available if the write region fails. Updates may take a few seconds to propagate.

## 13. Autoscaling

Redirect Service scales using:

- origin RPS
- active connections
- P99 latency
- CPU and memory
- L1 miss rate
- Redis latency
- database-pool utilisation

Management Service scales using creation and management traffic.

Analytics consumers scale using broker lag and oldest-event age.

## 14. Observability

Monitor:

- redirect and creation P50/P95/P99
- CDN, L1, and Redis hit ratios
- datastore fallback ratio
- generated-code collision retries
- custom-alias conflicts
- cache stampede count
- hot-link origin traffic
- replication lag
- cache invalidation delay
- lifecycle outbox backlog
- analytics event loss and consumer lag
- PostgreSQL pool saturation
- Redis latency and availability

## 15. Trade-offs

### 301 vs 302

`301` gives better caching but is harder to change. `302` gives more lifecycle control and is the default.

### Random Base62 vs encoded ID

Random codes are harder to enumerate but need collision retries. Encoded IDs are collision-free but predictable.

### Availability vs immediate global consistency

Regional replication gives low-latency reads but allows a small staleness window after updates.

### Analytics completeness vs redirect availability

Redirects continue even when analytics is degraded.

### Long TTL vs invalidation speed

Long TTLs improve cache efficiency but increase stale-response risk.
