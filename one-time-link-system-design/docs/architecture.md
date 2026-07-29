# Detailed Architecture

## 1. API surface

### Management APIs

```http
POST   /v1/one-time-links
GET    /v1/one-time-links/{link_id}
GET    /v1/one-time-links?cursor=...
POST   /v1/one-time-links/{link_id}/revoke
```

### Public APIs

```http
GET  /v/{token}
POST /v/{token}/consume
```

`GET` returns a non-consuming landing page. `POST` performs the state-changing consume operation.

## 2. Creation path

1. Authenticate and authorise the tenant user.
2. Validate payload, expiration, passcode policy, quotas, and rate limits.
3. Enforce tenant-scoped idempotency.
4. Generate a 256-bit random token.
5. Store only `HMAC-SHA-256(server_pepper, raw_token)`.
6. Encrypt inline content with AES-256-GCM using a per-link DEK.
7. Wrap the DEK through KMS.
8. In one PostgreSQL transaction:
   - enforce tenant quota
   - insert the link
   - persist idempotency state
   - increment active-link usage
   - insert an audit outbox event
9. Return `201 Created` with the public URL.

The raw token is never stored in plaintext.

## 3. Consumption path

1. `GET /v/{token}` returns a reveal page without consuming.
2. Recipient calls `POST /v/{token}/consume`.
3. Service hashes the token, applies rate limits, checks Redis, and verifies optional passcode.
4. PostgreSQL atomically transitions `ACTIVE → CONSUMED`.
5. Only the winning request receives payload metadata through `RETURNING`.
6. Commit the short database transaction.
7. After commit, unwrap the DEK through KMS and decrypt or stream the object.
8. Return content with `Cache-Control: no-store` and `Referrer-Policy: no-referrer`.

If delivery fails after commit, the link remains consumed.

## 4. Revocation

```sql
UPDATE one_time_links
SET status = 'REVOKED',
    revoked_at = now(),
    revoked_by_user_id = :user_id,
    version = version + 1
WHERE link_id = :link_id
  AND tenant_id = :tenant_id
  AND status = 'ACTIVE'
  AND expires_at > now()
RETURNING link_id;
```

Consumption and revocation both require `ACTIVE`, so only one can win.

## 5. Expiry

Consumption always checks `expires_at > now()`, so correctness does not depend on the sweeper.

The expiry sweeper later:
- marks records `EXPIRED`
- releases active-link quota
- emits audit events
- schedules payload cleanup

## 6. Core data model

```text
one_time_links
--------------
link_id
tenant_id
created_by_user_id
token_hash
idempotency_key
request_hash
status
content_type
payload_reference
content_size_bytes
encrypted_data_key
encryption_nonce
authentication_tag
kms_key_version
passcode_hash
display_name
created_at
expires_at
consumed_at
revoked_at
expired_at
revoked_by_user_id
consumed_request_id
version
```

Important constraints:

```sql
UNIQUE (token_hash)
UNIQUE (tenant_id, idempotency_key)
```

## 7. Redis and hot-token protection

Redis is a performance optimisation only.

Safe uses:
- negative cache for unavailable token digests
- per-IP rate limiting
- per-token rate limiting
- passcode failure counters
- optional short-lived `SET NX` hot-token gate
- token-to-shard or token-to-region routing metadata

If Redis fails, requests safely fall back to PostgreSQL.

## 8. Scaling

Separate:
- Public Consumption Service
- Authenticated Management Service
- Optional Object Streaming Gateway

All are stateless and horizontally scalable.

Use bounded database connection pools and PgBouncer.

Consumption, creation, and revocation use the PostgreSQL primary. Listing and dashboards may use read replicas.

Start with one highly available PostgreSQL cluster. If required later:

```text
shard_id = hash(token_hash) % N
```

Maintain an eventually consistent tenant-oriented read projection for listing.

## 9. Multi-region correctness

Each token has one authoritative owner region.

Requests may enter at the nearest edge but must be forwarded to the owner region for the atomic state transition.

If the owner region is unavailable, the initial design returns `503` rather than risking double disclosure.

Future failover requires:
- one new owner
- an ownership epoch
- fencing of the old region
- strongly coordinated replication

## 10. Security

- 256-bit random public tokens
- no plaintext token storage
- TLS everywhere
- AES-256-GCM envelope encryption
- KMS least-privilege access
- Argon2id passcode hashing
- generic public error responses
- per-IP and per-token rate limits
- DDoS and bot protection
- `GET` does not consume
- no third-party content on the landing page
- no secrets in logs or traces
- private object storage

## 11. Observability

Monitor:
- request rate and P95/P99 latency
- successful consumes
- conditional-update conflicts
- hot-token bursts
- unavailable-token traffic
- passcode failures
- PostgreSQL query latency and pool saturation
- lock waits and WAL generation
- Redis availability and hit rate
- KMS latency and failures
- object-storage failures
- audit outbox age
- expiry backlog

## 12. Main trade-offs

### Strict single-use vs guaranteed delivery
Consumption commits before delivery. This guarantees at-most-one disclosure but may consume the link even if delivery later fails.

### Proxy vs signed URL
Proxying preserves strict control. Signed URLs scale better but remain reusable until expiry.

### Redis vs PostgreSQL
Redis reduces load. PostgreSQL remains authoritative.

### Regional availability vs global correctness
Single-owner routing may return `503` during a regional outage. Active-active writes require globally coordinated consistency.
