Async order processing design for ShowTime (Part A)

1) Why Async?
- Synchronous payment calls (200–2000ms) hold DB connections and application threads, causing DB pool exhaustion under flash sale bursts.
- Example math (connection pool collapse): using the same numbers from CONCURRENCY.md, long payment hold times reduce max sustainable RPS to a few thousand. Offloading payments to workers keeps API response time low (~50ms) and minimizes DB-held time.

2) SQS Message Format (exact JSON)

{
	"bookingId": "<uuid>",
	"userId": "<uuid>",
	"eventId": <event_id>,
	"seatIds": [<seat_id>, ...],
	"totalAmount": 1234.56,
	"paymentToken": "<gateway_token>",
	"idempotencyKey": "<uuid-or-client-generated>",
	"createdAt": "2026-05-12T12:00:00Z"
}

Field rationale
- `bookingId`: booking UUID created by client or server for idempotent retries.
- `userId`: to send confirmations and for audit logs.
- `eventId`, `seatIds`: worker can avoid extra DB lookups for seat list; still verify before final commit.
- `totalAmount`: double-check for tampering; worker should validate amount against DB pricing.
- `paymentToken`: gateway token/nonce required to charge.
- `idempotencyKey`: dedupe retries at worker/payment gateway.
- `createdAt`: useful for visibility/monitoring and debugging.

3) Payment Worker Logic (numbered steps)

On message receive:
1. Parse message and check local idempotency store (e.g., `payments_processed:{idempotencyKey}` in Redis) — if processed, ACK and delete message.
2. Read booking row from DB (`SELECT * FROM bookings WHERE id = bookingId`). If booking not `pending`, ACK and delete message.
3. Optionally re-validate seat availability atomically: begin DB transaction, check seats for seatIds are held by booking or `status='held'` with `held_by=booking.user_id`; if mismatch, set booking `failed`, release holds, commit, send failure notification, delete message.
4. Call payment gateway with `paymentToken` and `idempotencyKey` (HTTP call).
5a. On payment SUCCESS: within a DB transaction:
		- UPDATE bookings SET status='confirmed', payment_reference=..., updated_at=now() WHERE id=bookingId;
		- UPDATE seats SET status='booked', held_by=NULL, held_until=NULL, version = version + 1 WHERE id = ANY(seatIds);
		- INSERT rows into booking_seats if not already present (idempotent upsert).
		- commit transaction; send confirmation email/SMS; delete SQS message.
5b. On payment FAILURE: within a DB transaction:
		- UPDATE bookings SET status='failed', payment_reference=... WHERE id=bookingId;
		- UPDATE seats SET status='available', held_by=NULL, held_until=NULL WHERE id = ANY(seatIds);
		- commit transaction; send failure notification; delete SQS message.

4) Edge cases
- API server crashes after publishing to SQS but before responding: the booking was created and the SQS message persisted. The payment worker will process the message and the user will receive confirmation via SMS/email despite the client timeout. The API must be idempotent so retries do not create duplicate bookings.
- Payment gateway times out (no success/no failure): worker should treat as transient and retry up to configured attempts (see SQS receive count). Use idempotencyKey with gateway. If retries exceed threshold, move message to DLQ for manual inspection.

5) SQS configuration
- Visibility timeout: 120 seconds. Rationale: set to ~2× expected maximum payment process time (network + gateway latency + DB commits) so that a single worker has time to finish before message becomes visible again.
- MaxReceiveCount before DLQ: 3. Rationale: allow transient gateway/network retries but route repeat failures to DLQ for human investigation.
- Dead Letter Queue (DLQ): monitored by alarms; manual triage for stuck bookings.

Idempotency and monitoring
- Maintain a `payments_processed` idempotency store in Redis or a DB table keyed by `idempotencyKey` to ensure exactly-once processing semantics.
- Emit CloudWatch / CloudMetrics events for message age, DLQ count, and payment success/failure rate.

Security and retries
- Do not store full card data in SQS. Use gateway tokens/nonces only. Ensure SQS encryption at rest and TLS in transit.
