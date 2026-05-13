Post-Roast Design Updates

## Update 1: Per-user seat hold counter

**Triggered by:** Panel Question 3 - one user holding 200 seats through multiple tabs.

**What changed:**
- Added Redis counter key `holds:{userId}:count`.
- Seat hold requests now check the counter before lock acquisition.
- If the count is 8 or higher, the API returns `429 Too Many Requests`.
- The counter is incremented on successful hold and decremented when the hold is released or the booking is confirmed.
- The hold window for active sale periods is reduced from 10 minutes to 3 minutes to release abandoned seats faster.

**Why this is necessary:**
The original design prevented double-booking, but it did not prevent a single user from tying up too much inventory. This update reduces abusive hoarding without adding DB load.

**What it costs:**
One extra Redis lookup and counter write on the seat-hold path, plus slightly more application logic around hold release.

**What it still doesn't solve:**
It does not stop a coordinated attacker using many accounts or many payment instruments.

## Update 2: SQS publish circuit breaker with fallback booking path

**Triggered by:** Panel Question 4 - SQS outage during an active sale.

**What changed:**
- Added a circuit breaker around SQS publish.
- If SQS publish fails continuously for 60 seconds during a sale window, the API switches to a reduced-throughput synchronous booking path.
- Added/relied on the booking status polling endpoint `GET /bookings/:id` so the client can keep showing `pending` while the system recovers.
- CloudWatch alarms are tied to queue publish failures and queue depth.

**Why this is necessary:**
The original design assumed SQS stayed available. That is acceptable for normal operation, but an extended outage would otherwise leave the API claiming progress without any work actually happening.

**What it costs:**
More code paths, more operational monitoring, and a slower emergency fallback when the queue is unhealthy.

**What it still doesn't solve:**
The synchronous fallback still reduces throughput and can reintroduce connection pressure; it is an emergency path, not the steady-state design.

