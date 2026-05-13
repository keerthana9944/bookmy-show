ShowTime / BookMyShow - Full System Architecture

```text
Mobile / Browser  ← Part A: QUEUE.md (poll /bookings/:id after 202)
			│
			▼
CloudFront CDN  ← Part A: CACHE.md (static assets + cached event pages)
	- serves: JS/CSS/images, cached event detail pages
	- passes through: /api/* to origin on cache miss
			│
			▼
Application Load Balancer (ALB)  ← Part A: CONCURRENCY.md / QUEUE.md
	- SSL termination
	- health checks every 10s
	- rate limit rule: 200 req/IP/min
			│
			▼
Node.js API Auto-Scale Group (4–20 instances)  ← Part A: hybrid locking + async queue
	- reads: Redis cache, Redis hold counter, replica reads for browsing
	- writes: PostgreSQL primary for booking transactions
	- publishes: booking messages to SQS
			│
			├──────────────────────────────────────────────────────────────────┐
			│                                                                  │
			▼                                                                  ▼
Redis Cluster (ElastiCache, 3 nodes)  ← Part A: SETNX hot-path lock      SQS Payment Queue  ← Part A: QUEUE.md
	- cache:event:{id} TTL 3600s                                           - message: bookingId,userId,eventId,seatIds,
	- cache:availability:{event}:{cat} TTL 30s                               totalAmount,paymentToken,idempotencyKey
	- seat_lock:{event}:{seat} SETNX TTL 5s                                   - visibility timeout: 120s
	- holds:{userId}:count TTL 15m  [Update 1: anti-abuse]                   - max receive count: 3
			│                                                                    - DLQ: payment-dlq
			│                                                                    - circuit breaker fallback: API sync path [Update 2]
			│                                                                    │
			│                                                                    ▼
			│                                                        ECS Fargate Payment Workers ×10  ← Part A: async order flow
			│                                                          1) read SQS message
			│                                                          2) call payment gateway
			│                                                          3) write booking/payment status to DB primary
			│                                                          4) publish notification event to SNS
			│                                                          5) delete SQS message
			│                                                                    │
			│                                                                    ▼
			│                                                           External Payment Gateway  ← Part A: QUEUE.md (payment provider call)
			│
			├── READ / CACHE MISS ───────────────────────────────────────────────────────────────────────────────┐
			│                                                                                                    │
			▼                                                                                                    ▼
PostgreSQL Primary (RDS db.r6g.xlarge)  ← Part A: SCHEMA.md (source of truth)                     PostgreSQL Read Replicas ×2
	- writes: booking creation, seat hold/confirm/release, payment status                              ← Part A: browsing reads only
	- seat version checks for optimistic concurrency                                                    - event details
	- row-level source of truth for booking seats                                                        - seat maps
			│                                                                                                 - user booking history
			└────────────────────────────── replication ──────────────────────────────► Read Replica 1
																																								└► Read Replica 2

SNS  ← Part A: QUEUE.md (booking confirmed event)
	├── SES Email
	└── SMS (SNS / Twilio integration)
```

Data flow labels

- Browser → CloudFront: static assets, cached event pages, API requests on miss
- CloudFront → ALB: dynamic /api/* calls
- ALB → Node.js API: HTTPS requests, routed by path
- API → Redis: availability counts, seat locks, per-user hold counters
- API → PostgreSQL primary: booking writes, seat holds, confirmations, release transactions
- API → Read replicas: event detail, seat map, and booking history reads
- API → SQS: payment job payloads for async processing
- SQS → Worker: bookingId + paymentToken + seatIds + idempotencyKey
- Worker → Payment gateway: external charge request
- Worker → PostgreSQL primary: confirm/fail booking, book/release seats
- Worker → SNS: booking confirmed event for email/SMS

Part A annotations back to the design

- Redis SETNX lock: justified in `docs/CONCURRENCY.md` because PostgreSQL FOR UPDATE saturates at about 2,840 RPS under the given pool model.
- Async queue: justified in `docs/QUEUE.md` because synchronous payment would hold DB connections long enough to exhaust the pool during the flash sale.
- Read replicas: justified in `docs/SCHEMA.md` and `docs/CACHE.md` because browsing reads can tolerate staleness while bookings must always hit the primary.

Updated architecture after panel feedback

- [Update 1] Per-user seat hold counter in Redis: `holds:{userId}:count` limits abusive multi-tab seat hoarding to 8 concurrent holds.
- [Update 2] SQS publish circuit breaker: after 60 seconds of publish failure, API falls back to a reduced-throughput synchronous booking path and shows booking status via polling.

