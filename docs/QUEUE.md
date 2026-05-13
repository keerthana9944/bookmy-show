**Async Order Processing: SQS message format, worker flow, and failure handling**

Why Async?
---------
- Payment gateway calls are slow (200–2,000ms). If API waits synchronously, DB connections remain held for the duration, rapidly exhausting the pool during a sale.
- By holding seats and returning 202 ASAP (publish to SQS), API latency stays <500ms and DB hold time is short (~10–50ms).

SQS message format (JSON)
-------------------------
{
	"bookingId": "<uuid>",
	"userId": "<uuid>",
	"eventId": <int>,
	"seatIds": [<int>,...],
	"totalAmount": 0.00,
	"paymentToken": "<opaque_token>",
	"idempotencyKey": "<uuid-or-client-generated>",
	"createdAt": "2026-05-13T12:00:00Z"
}

Field rationale
- `bookingId`: primary key for booking and idempotency — worker can lookup booking by id without creating duplicates.
- `userId`: to send notifications and for audit.
- `eventId`, `seatIds`: worker must confirm seats and update DB to `booked` on success.
- `totalAmount`: double-check amount against payment provider and prevent tampering.
- `paymentToken`: tokenized payment method (PCI obligations) used by worker to charge gateway.
- `idempotencyKey`: ensures repeated delivery or retries are safe.

Worker logic (numbered steps)
-----------------------------
1) Receive message from SQS (message becomes invisible per visibilityTimeout).
2) Parse JSON and validate required fields.
3) Idempotency check: if booking.status == 'confirmed' → delete message, return SUCCESS.
4) Call payment gateway with `paymentToken` and `idempotencyKey`.

	 Success path:
	 a) Begin DB transaction
	 b) UPDATE bookings SET status='confirmed', payment_ref=<gateway_ref>, confirmed_at=NOW() WHERE id=bookingId;
	 c) UPDATE seats SET status='booked', held_by = NULL, held_until = NULL WHERE id IN (seatIds);
	 d) Commit
	 e) Send notifications (email/SMS)
	 f) Delete SQS message

	 Failure path (payment failure):
	 a) Begin DB transaction
	 b) UPDATE bookings SET status='failed', payment_ref=<gateway_error> WHERE id=bookingId;
	 c) UPDATE seats SET status='available', held_by=NULL, held_until=NULL WHERE id IN (seatIds);
	 d) Commit
	 e) Notify user of failure
	 f) Delete SQS message

Visibility timeout, retries, DLQ
--------------------------------
- Visibility timeout: 120 seconds. Justification: payment calls can be slow and gateways sometimes retry/timeout; 2 minutes gives room for retries or third-party latency while preventing long invisible messages.
- MaxReceiveCount before DLQ: 3. After 3 failed processing attempts, message moves to DLQ for human intervention and alerting.

Edge cases
----------
- API server crashes after publishing to SQS but before responding: booking, seats and SQS message already exist — worker will process the message and user receives confirmation; the UI should poll booking status and handle eventual confirmation.
- Payment gateway timeout (no definitive success/failure): worker should follow idempotent retry logic — treat as transient and retry up to configured attempts; each retry uses same `idempotencyKey` so gateway does not double-charge. If retries exhausted, move message to DLQ and mark booking 'failed' after manual or automated reconciliation.

Idempotency and at-least-once delivery
--------------------------------------
- SQS guarantees at-least-once delivery. Worker must design for duplicate messages. The booking record + `idempotencyKey` + booking.status checks make processing idempotent.

Monitoring and alarms
---------------------
- CloudWatch (or equivalent) alarms for DLQ depth, worker error rate, queue latency. Pager alerts for >5 DLQ messages in 10 minutes.

