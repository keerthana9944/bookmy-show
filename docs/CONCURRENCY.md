**Concurrency Strategy — Options, math, and final choice**

Overview
--------
This document analyzes two practical choices for preventing double-booking under extreme contention:

- Option A — PostgreSQL row-level locks (SELECT FOR UPDATE)
- Option B — Redis distributed locks (SETNX with TTL)

Both are viable; the recommended production approach for the Hot-Path (ticket sale spike) is a Hybrid:
- Use Redis SETNX per-seat to protect the hot acquisition path (milliseconds),
- Persist a long-lived hold in PostgreSQL (held_until) and use optimistic version checks at final commit.

Option A — PostgreSQL SELECT FOR UPDATE
----------------------------------------
Transaction (single-seat, simplified):

BEGIN;
-- 1) Lock seat row(s)
SELECT id, status FROM seats
WHERE id = ANY($1::int[])
	AND status = 'available'
	FOR UPDATE;

-- 2) Update to held
UPDATE seats SET status = 'held', held_until = NOW() + INTERVAL '10 minutes', held_by = $2
WHERE id = ANY($1::int[]);

-- 3) Insert booking (pending)
INSERT INTO bookings (id, user_id, event_id, total_amount) VALUES ($3,$4,$5);

COMMIT;

How it prevents double-booking: FOR UPDATE takes a row lock. Competing transactions queue at DB level.

Capacity math (pool exhaustion)
- Assumptions used here:
	- max_connections (PgBouncer pooled) = 500
	- non-payment RPS fraction = 80%, avg non-payment DB hold = 20ms (0.02s)
	- payment RPS fraction = 20%, avg payment DB hold = 800ms (0.8s)

Connections held = RPS * (0.8*0.02 + 0.2*0.8) = RPS * 0.176s
Pool exhaustion when connections held >= 500 → RPS ~= 500 / 0.176 ≈ 2,840 RPS

At 2,840 RPS the DB pool is saturated. A coordinated flash of 500,000 parallel UI taps creates orders of
magnitude higher instantaneous RPS and will overwhelm a FOR UPDATE approach unless you horizontally scale DB
and dramatically reduce per-request DB hold time (which violates budget).

Deadlocks and multi-seat bookings
- If a user books multiple seats in one transaction and two users lock seats in different orders, DB deadlock can occur.
	Mitigation: canonical ordering of seat ids (always lock seats in ascending id order), retry transaction on deadlock, or use application-level orchestration.

Option B — Redis SETNX Distributed Lock
---------------------------------------
Pattern (per seat):

const lockKey = `seat_lock:${eventId}:${seatId}`
const lockValue = `${instanceId}:${Date.now()}`
// Acquire
redis.set(lockKey, lockValue, 'NX', 'EX', lockTTLSeconds)

// Release (atomic):
-- Lua script
if redis.call('get', KEYS[1]) == ARGV[1] then
	return redis.call('del', KEYS[1])
else
	return 0
end

Why Redis helps:
- Sub-millisecond SETNX ops, high ops/sec throughput (100k+/s on modest cluster), avoids holding DB connections during acquisition.

Failure modes & TTL choice
- If Redis dies and locks are unavailable, the system must fall back to a safe DB-based path (e.g., SELECT FOR UPDATE) or fail-open with conservative error.
- Lock TTL must be long enough to complete the DB update and short enough to avoid long blocking from abandoned locks.
	- Chosen default for acquisition lock TTL: 5s
	- Justification: Typical seat hold DB writes <50–200ms; 5s gives leeway for network jitter; worker can renew lock if needed.

Hybrid choice and justification
--------------------------------
- Choose Hybrid (Redis SETNX for hot-path acquisition; PostgreSQL persistent hold + optimistic versioning for final confirmation).

Why hybrid?
- Performance: Redis absorbs the flash and provides very fast mutual exclusion without DB connection locking.
- Correctness: PostgreSQL remains the source-of-truth; optimistic versioning + transactional updates guarantee final consistency.
- Budget: A small ElastiCache cluster (2–3 nodes r6g.large) fits the $2,000/mo budget in the cost plan, and avoids needing a massively scaled RDS tier.

What this cannot handle
- If demand exceeds Redis cluster capacity (extreme beyond planned sizing), locking latency increases and failures rise. If database primary is unreachable during final commit, bookings may remain 'pending' until manual reconciliation.

When to switch
- If steady-state peak RPS grows beyond ~100k acquisition attempts/sec per event, consider moving to a dedicated seat-partitioned locking service (sharded Redis cluster or scalable consensus like etcd) and increasing DB replicas and write throughput.

