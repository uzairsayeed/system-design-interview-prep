# Data Model

## `jobs`

| Column | Purpose |
|---|---|
| `job_id` | Public immutable job identifier |
| `tenant_id` | Tenant ownership |
| `idempotency_key` | Tenant-scoped request identity |
| `request_hash` | Detect conflicting idempotency-key reuse |
| `status` | Current authoritative job state |
| `priority` | HIGH, MEDIUM, or LOW |
| `cpu_millis` | Requested CPU |
| `memory_mb` | Requested memory |
| `gpu_count` | Requested GPU count |
| `gpu_type` | Optional required GPU model |
| `spec_json` | Flexible execution specification |
| `attempt_count` | Number of attempts created |
| `max_attempts` | Total permitted attempts |
| `next_eligible_at` | Retry scheduling time |
| `current_attempt_id` | Current authoritative attempt |
| `version` | Optimistic concurrency control |
| `created_at` | Creation time |
| `updated_at` | Last state change |

Important constraint:

```sql
UNIQUE (tenant_id, idempotency_key)
```

Suggested scheduling index:

```sql
CREATE INDEX jobs_schedulable_idx
ON jobs (status, next_eligible_at, priority DESC, created_at);
```

## `job_attempts`

| Column | Purpose |
|---|---|
| `attempt_id` | Unique execution attempt |
| `job_id` | Parent job |
| `attempt_number` | 1-based attempt number |
| `worker_id` | Assigned worker |
| `status` | ASSIGNED, RUNNING, SUCCEEDED, LOST, FAILED, CANCELLED |
| `lease_id` | Fencing token |
| `lease_expires_at` | Ownership expiry |
| `started_at` | Container start confirmation |
| `finished_at` | Terminal timestamp |
| `failure_type` | USER, INFRASTRUCTURE, TIMEOUT, CANCELLED |
| `failure_code` | Machine-readable failure |
| `failure_details` | Diagnostic payload |
| `output_uri` | Attempt-scoped output |

Suggested lease-reaper index:

```sql
CREATE INDEX attempts_expired_lease_idx
ON job_attempts (lease_expires_at)
WHERE status IN ('ASSIGNED', 'RUNNING');
```

## `workers`

| Column | Purpose |
|---|---|
| `worker_id` | Worker identity |
| `pool` | CPU, A100, H100, etc. |
| `status` | ACTIVE, DRAINING, UNHEALTHY |
| `total_cpu_millis` | Total CPU |
| `allocated_cpu_millis` | Authoritative allocation |
| `total_memory_mb` | Total memory |
| `allocated_memory_mb` | Authoritative allocation |
| `total_gpu_count` | Total GPUs |
| `allocated_gpu_count` | Authoritative GPU allocation |
| `gpu_type` | Worker GPU model |
| `last_heartbeat_at` | Latest liveness report |
| `version` | Conditional reservation version |

## `tenant_quotas`

Defines configured maximum concurrent resources per tenant.

## `tenant_resource_usage`

Stores current authoritative resource usage. Kept separate from quota configuration to simplify atomic updates.

## `outbox_events`

| Column | Purpose |
|---|---|
| `event_id` | Unique publication identity |
| `aggregate_id` | Job ID |
| `event_type` | `JOB_READY` |
| `payload` | Lightweight queue payload |
| `created_at` | Event creation |
| `published_at` | Successful publish time |
| `retry_count` | Publisher attempts |
