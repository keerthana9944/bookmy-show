ShowTime Design Decisions

## Decision: Hybrid concurrency strategy

**Context:** 5 lakh users can hit the same seat at once, but the system also has a strict budget and a hard no-double-booking requirement.

**Options considered:**
1. PostgreSQL `SELECT FOR UPDATE` only - simplest correctness model, but the DB pool becomes the bottleneck under flash-sale load.
2. Redis `SETNX` only - fastest hot path, but too risky without a durable DB source-of-truth and final consistency check.
3. Hybrid Redis + PostgreSQL - Redis protects the hot acquisition path; PostgreSQL enforces the final transaction and version check.

**Why chosen:**
The pool model in `docs/CONCURRENCY.md` shows synchronous DB locking saturating around 2,840 RPS, which is too low for the launch event. Redis absorbs contention cheaply while PostgreSQL keeps the system correct, and the combined infrastructure fits the roughly $2,000/month budget.

**Tradeoffs accepted:**
Redis failure reduces throughput and can cause temporary 503s, and the system is more complex than a DB-only design. The design also still needs careful lock TTL tuning to avoid hold leakage.

**Revision trigger:**
If steady-state acquisition demand grows beyond the planned Redis cluster capacity or if the booking transaction pattern becomes more complex than seat-level locking, move to sharded lock management or a dedicated coordination service.

## Decision: Event-driven cache invalidation with TTL fallback

**Context:** Availability counts and event pages are read far more often than they are written, but seat state must not go stale for long.

**Options considered:**
1. TTL-only cache expiry - simplest, but stale counts could persist for too long after a seat changes.
2. Event-driven invalidation plus TTL - delete the touched key on write, then let the next read repopulate it.

**Why chosen:**
Availability is user-facing and must be reasonably fresh. Targeted invalidation keeps the read path fast while removing the worst stale-data window. The 30-second TTL remains as a backstop when invalidation is missed.

**Tradeoffs accepted:**
Each seat status change adds one more Redis write, and availability may still be slightly stale between the write and the next cache refresh.

**Revision trigger:**
If write volume becomes high enough that invalidation chatter dominates Redis cost, move to precomputed counters or a streaming aggregate model.

## Decision: UUID for booking identifiers

**Context:** Booking IDs must be safe to expose to clients and useful for retries after network failures.

**Options considered:**
1. SERIAL / auto-increment - easy, but predictable and enumerable.
2. UUID v4 - unguessable, safe for public exposure, and can be generated before DB insert for idempotency.

**Why chosen:**
UUIDs prevent booking enumeration and make retry handling simpler because the client can reuse the same booking ID on repeat submissions.

**Tradeoffs accepted:**
UUID indexes are larger than integer keys and slightly less cache-friendly than SERIAL.

**Revision trigger:**
If storage footprint or secondary index cost becomes a material issue, consider time-ordered UUIDs or a hybrid public ID/internal integer ID model.

## Decision: SQS visibility timeout at 120 seconds

**Context:** The payment worker calls an external payment gateway, and the queue must survive slow or retried calls without duplicate processing.

**Options considered:**
1. Short timeout (20-30s) - faster redelivery, but too aggressive for slow payment processing.
2. Longer timeout (120s) - gives the worker enough time for retries and temporary gateway delays.

**Why chosen:**
120 seconds fits the slowest realistic payment paths while keeping the message from being invisible for too long. The worker still needs idempotency keys to handle retries safely.

**Tradeoffs accepted:**
Messages may remain invisible for longer if a worker hangs, and the system still needs DLQ monitoring for poisoned messages.

**Revision trigger:**
If measured payment latency falls materially, shorten the visibility timeout to improve retry responsiveness; if latency rises, move to heartbeat-based visibility extension.

## Decision: Per-user hold counter and hold-window reduction

**Context:** A single user can abuse tabs or sessions to hoard too many seats during a flash sale.

**Options considered:**
1. IP rate limiting only - too weak; one user can bypass it.
2. Redis per-user hold counter with a cap and shorter peak-sale hold TTL - directly constrains seat hoarding.

**Why chosen:**
It addresses the actual abuse pattern with a low-latency Redis check before seat lock acquisition, and it is cheap enough to operate inside the budget.

**Tradeoffs accepted:**
It does not stop a determined attacker using many accounts.

**Revision trigger:**
If abuse shifts to account farming, add stronger identity verification or payment-method preauthorization.

