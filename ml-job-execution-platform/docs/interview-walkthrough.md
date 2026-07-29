# Interview Walkthrough

## Recommended structure

### 1. Clarify requirements

Ask about:

- Job specification
- CPU, memory, and GPU requirements
- Submission and concurrency scale
- Priorities
- Tenant quotas
- Retry policy
- Cancellation
- Read consistency
- Region scope
- Autoscaling
- Duplicate execution tolerance

### 2. State the high-level architecture

Keep the first pass simple:

```text
Client
→ Job API
→ PostgreSQL + Outbox
→ Priority Queues
→ Scheduler
→ Worker Control API
→ Worker Pools
→ Object Storage
```

Then say:

> I will now deep-dive into submission durability, atomic scheduling, worker execution, and failure recovery.

### 3. Submission path

Explain:

- Tenant-scoped idempotency
- Commit before `201`
- Transactional outbox
- At-least-once publication

### 4. Scheduler path

Explain:

- Weighted priority
- Tenant fairness
- Redis candidate shortlist
- PostgreSQL atomic assignment transaction
- Duplicate scheduler protection

### 5. Worker path

Explain:

- Long-poll assignment
- Attempt and lease
- Heartbeat renewal
- Dedicated terminal callbacks
- Attempt-specific output

### 6. Failure recovery

Explain:

- Lease reaper
- Retry promoter
- New attempt and new lease
- Stale-worker fencing
- Maximum attempts

### 7. Close with scaling and trade-offs

Mention:

- Control-plane HPA
- Resource-specific worker autoscaling
- Queue age and unschedulable demand
- At-least-once execution
- Redis speed versus PostgreSQL authority
- Worker pull versus push

## Two-minute summary

> The platform uses an asynchronous architecture. The Job API authenticates and validates a tenant request, enforces idempotency, and in one PostgreSQL transaction creates the pending job and a transactional-outbox event. It returns success only after commit.
>
> The outbox publisher sends scheduling events to priority queues. A horizontally scaled scheduler applies weighted priority and tenant fairness. It uses Redis to shortlist compatible workers, while PostgreSQL atomically confirms the job is pending, reserves tenant and worker resources, creates an attempt and lease, and marks the job assigned.
>
> Workers long-poll for assignments and send periodic heartbeats to renew leases, report free capacity, and receive control commands. Completion is accepted only from the current attempt and lease holder. Each attempt writes to an isolated output path.
>
> If a worker disappears, a lease reaper marks the attempt lost, releases reservations, and applies the retry policy. Eligible retries return to pending through a retry promoter. This provides at-least-once execution, so temporary duplicate computation is possible, but stale attempts cannot commit state or promote output.
>
> Control-plane services scale horizontally, while worker pools scale from pending resource demand, queue age, utilisation, and provisioning-in-flight. The primary trade-off is accepting duplicate computation to avoid losing accepted work.
