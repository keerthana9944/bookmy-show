**Cache Design: keys, TTLs, invalidation rules**

Goals
-----
- Reduce RDS read pressure during flash sales
- Ensure counts and static data are fast to read
- Never rely on cache for per-seat canonical availability

Cache entries
-------------

1) Event details
- Key: `event:{event_id}`
- Value: JSON blob { id, name, venue_id, starts_at, status, total_seats, metadata }
- TTL: 3600s (1 hour)
- Invalidate: on event UPDATE (status change or metadata change) — application deletes the key after DB commit.
- Why: event metadata changes very infrequently; 1h TTL reduces DB hits and is acceptable for occasional updates.

2) Seat availability count per event + category
- Key: `availability:{event_id}:{category}`
- Value: integer count (stringified)
- TTL: 30s (cache-aside) with targeted invalidation
- Invalidate: when any seat for that event+category changes status (available→held/booked/released) the worker deletes the key.
- Why 30s: balances freshness and throughput — availability is user-facing and should be near-real-time; 30s gives reduced DB load but still low staleness. Targeted invalidation makes counts effectively immediate on writes.

3) Static seat map layout
- Key: `seatmap:{event_id}`
- Value: JSON layout {sections, rows, seat_counts, categories}
- TTL: 86400s (24 hours)
- Invalidate: on event cancellation or seatmap change (rare)
- Why: static for a given event; cheap to cache long-term.

What NOT to cache
------------------
- Individual seat status (e.g., seat A-12 available/booked): NEVER cached as authoritative. Use Redis locks for acquisition and DB as source-of-truth for final state. Caching per-seat status creates race windows and potential double-booking.

Invalidation strategy
---------------------
- Use cache-aside with targeted deletion (delete keys touched) on successful DB commit.
- Example flow on seat status change (pseudocode):

function updateSeatStatus(seatId, eventId, category, newStatus) {
	BEGIN DB TRANSACTION
		UPDATE seats SET status = newStatus WHERE id = seatId;
	COMMIT

	// targeted invalidation
	redis.del(`availability:${eventId}:${category}`)
}

Why delete (vs update-in-place)?
- Simpler and safer: computing availability is a small DB COUNT; delete avoids complex race conditions. For high write volume, an incremental counter update could be used but requires careful synchronization and compensating logic.

Read flow (cache-aside)
-----------------------
1) Try Redis.get(key)
2) If hit, return parsed value
3) If miss, SELECT from DB, redis.setex(key, TTL, value)

Notes on write load and cost
---------------------------
- Targeted invalidation adds Redis writes on every seat status change, but this is cheaper than DB reads from many UI clients. Under the budget, a small ElastiCache cluster handles the invalidation ops for a sale spike.

