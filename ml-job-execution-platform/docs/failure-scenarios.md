# Failure Scenarios

## Job API crashes before commit

No job was accepted. The client retries using the same idempotency key.

## Job API crashes after commit but before returning `201`

The retry reaches the unique idempotency constraint and returns the existing job.

## Outbox publisher crashes after broker publish

The outbox row may be published again. Duplicate queue delivery is expected and handled by the scheduler's conditional job claim.

## Two schedulers process the same job

Both may shortlist candidates. Only one can atomically change the job from `PENDING` to `ASSIGNED`. The loser rolls back its quota and worker reservation.

## Redis is stale

Redis only proposes candidates. PostgreSQL validates actual remaining capacity and worker version.

## Redis is unavailable

Running jobs remain unaffected. The scheduler can enter degraded mode with bounded PostgreSQL candidate queries or temporarily pause new assignments.

## PostgreSQL is unavailable

New submissions return `503`; scheduling pauses. Workers continue executing for a bounded grace period and retry heartbeat and terminal callbacks.

After recovery, the lease reaper should use a reconciliation grace period to avoid mass false retries.

## Worker crashes

Heartbeat renewal stops. The lease reaper marks the attempt lost, releases reservations, and applies retry policy.

## Worker is partitioned but still executing

A newer attempt may run elsewhere. The stale worker cannot commit state or promote output because its attempt and lease are no longer current.

## Worker receives assignment but never starts

The initial assignment lease is intentionally short. Lease expiry recovers the job.

## Cancellation command is not received

The job remains `CANCELLING`. Lease expiry and recovery eventually finalise cancellation according to policy.

## Poison job repeatedly crashes workers

A missed heartbeat alone is not enough to declare a poison job. Correlate:

- Exit status
- Node health
- Repeated failures across different workers
- Failure timing
- Container-runtime errors

Repeated immediate crashes across healthy workers may quarantine the job and mark it terminal.

## Object-store write succeeds but completion callback fails

The worker retries the idempotent completion callback. Output remains attempt-scoped until the current lease holder successfully promotes it.
