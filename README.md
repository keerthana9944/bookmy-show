# bookmy-show

## Constraints — 5L users, zero double-bookings, $2,000/month

This repository contains the Part A architecture deliverables for the ShowTime ticketing
system: schema, concurrency strategy, cache design, and async order queue.

Three hard constraints (the trilemma)

- 5,00,000 concurrent users at T+0 (peak demand at 12:00:00)
- 0 acceptable double-bookings (seat uniqueness must be enforced)
- $2,000/month AWS budget (must include ALB, RDS, ElastiCache, SQS, ECS/Fargate, small fleet of API servers)

How these constraints interact

- Speed vs Correctness: Ensuring zero double-bookings requires locking. Strong locking (DB FOR UPDATE) increases latency and ties up DB connections; at large scale this becomes the bottleneck. We therefore use a hybrid: Redis SETNX (fast, in-memory locks) for the hot acquisition path and PostgreSQL as the final source-of-truth with optimistic version checks.

- Scale vs Budget: The budget limits how many DB and cache nodes we can run. We optimize for cost by caching high-read items, using an async payment flow to avoid long-lived DB connections, and sizing instances conservatively (example cost plan in docs). The hybrid lock approach lets a small ElastiCache cluster absorb the flash rather than scaling the DB beyond budget.

- Correctness vs Budget: The architecture uses Redis to avoid DB connection saturation during flash sales but accepts the operational cost of a small Redis cluster (~3 nodes) because DB-only locking would require much larger RDS capacity and exceed budget.

Peak RPS estimate and bottleneck analysis

Assumptions:
- 500,000 users all attempt the same single-seat purchase within 1 second (worst-case spike). In practice client-side jitter spreads this over several seconds; we design for worst-case.
- Typical API call mix during sale: 80% non-payment (seat selection & hold calls), 20% payment submissions (publish to queue). Non-payment DB hold time ~20ms; payment hold time in sync approach ~800ms (if synchronous).

Pool exhaustion formula (connections held):

Connections held = RPS * (fraction_non_payment * avg_non_payment_time_s + fraction_payment * avg_payment_time_s)

With optimistic async payments (publish to queue) avg_payment_time_s becomes ~0.05s (DB hold only for creating pending booking + seat hold). Using conservative numbers:

- fraction_non_payment = 0.80, avg_non_payment_time = 0.02s
- fraction_payment = 0.20, avg_payment_time(sync) = 0.8s, avg_payment_time(async) = 0.05s
- max_connections (PgBouncer) = 500

Sync payments (bad):
Connections held per RPS = RPS * (0.8*0.02 + 0.2*0.8) = RPS * 0.176
Pool exhaust RPS ≈ 500 / 0.176 ≈ 2,840 RPS

Async payments (designed):
Connections held per RPS = RPS * (0.8*0.02 + 0.2*0.05) = RPS * 0.024
Pool exhaust RPS ≈ 500 / 0.024 ≈ 20,833 RPS

Interpretation: With async payments and short DB holds, RDS can handle ~20k RPS before exhausting connections. The flash of 500k concurrent taps must be absorbed at the API+Redis layer (fast acquisition locks, caching) and throttled or spread across more seconds — the combination of CDN, client-side jitter, and efficient Redis locking lets the system avoid immediate DB overload.

What the $2,000/month budget buys

Example allocation (approximate monthly cost):
- 6 × t3.xlarge EC2 (API): ~$720
- RDS primary db.r6g.xlarge: ~$262
- 2 × RDS read replicas db.r6g.large: ~$262
- ElastiCache r6g.large ×3 (small cluster): ~$359
- ECS/Fargate small workers: ~$073
- ALB + CloudFront + SQS + monitoring: ~$165
Total ≈ $1,840/mo (leaves buffer under $2,000)

What changes if budget = $500/mo

- Must reduce instance sizes and remove read replicas and shrink Redis to a single node. This forces a DB-first strategy (SELECT FOR UPDATE) for correctness at small scale, but reduces throughput drastically — realistic peaks must be limited (lower concurrent users) or rely on heavy traffic shaping and queuing at the edge.

Notes

- These constraints and calculations are summarized and expanded in `docs/CONCURRENCY.md`, `docs/CACHE.md`, `docs/SCHEMA.md`, and `docs/QUEUE.md`.
