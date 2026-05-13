Architecture Diagram (ASCII)

Mobile / Browser
			│
			▼
CloudFront CDN  ── Static assets & cached event pages (HTML/JSON) ── cache-hit -> served
			│
			│ (API calls forwarded, TLS to ALB)
			▼
Application Load Balancer (ALB)
	- SSL termination
	- Health check: /health every 10s
	- Rate limit: 200 req/IP/min
			│
			▼
Node.js API Auto-scale Group (4–20 instances)
	- Handles: /api/* HTTP
	- Reads: Redis cache keys (availability:event:cat, event:{id}, seatmap:{id})
	- Pre-checks: per-user hold counter in Redis (holds:{userId})
	- Writes: PostgreSQL primary (bookings, seat state)
	- Publishes: SQS payment messages
	- Polling endpoint: GET /bookings/:id -> reads booking.status from DB
			│
	┌───────┬────────┬─────────┐
	│       │        │         │
	│       │        │         │
	▼       ▼        ▼         ▼
READ   LOCKS/   WRITE      PUBLISH
to     COUNTERS to         to
Redis  (Redis)  Postgres   SQS
(cache)          (primary)  (payment-queue)

Redis Cluster (ElastiCache - 3 nodes)
	- Cache keys: availability:{event_id}:{category} (TTL 30s)
	- Event cache: event:{event_id} (TTL 3600s)
	- Seat map: seatmap:{event_id} (TTL 86400s)
	- Per-user hold counter: holds:{userId} (TTL 15m)
	- Seat locks: seat_lock:{event_id}:{seat_id} (SETNX, lockValue=userId:ts, TTL 5s)
	- Annotation: Redis used for hot-path locks and rate-limiting (see CONCURRENCY.md)

PostgreSQL Primary (RDS db.r6g.xlarge)
	- Writes: bookings, booking_seats, seats updates
	- Ensures ACID; version-based optimistic checks on `seats.version`
	- Replication -> Read Replica 1, Read Replica 2 (async)
	- Annotation: Primary is authoritative for final commits (see SCHEMA.md)

Read Replicas (2)
	- Serve: event details, seatmap, user booking history
	- Not used for seat write/lock checks
	- Accept eventual consistency (replica lag tolerated for UI only)

SQS Payment Queue
	- Message: { bookingId, userId, eventId, seatIds, totalAmount, paymentToken, idempotencyKey }
	- Visibility timeout: 120s
	- MaxReceiveCount: 3 -> DLQ: payment-dlq
	- Annotation: Async payments to avoid blocking DB connections (see QUEUE.md)

Payment Workers (ECS Fargate, autoscaled)
	- 1. Read message from SQS
	- 2. Check idempotency store (payments_processed:{idempotencyKey})
	- 3. Call payment gateway (Razorpay/PayU) with idempotencyKey
	- 4. On success: transactional DB update: bookings.status='confirmed', seats.status='booked', insert booking_seats
	- 5. Publish notification via SNS (email/SMS)
	- 6. Delete SQS message

SNS -> SES / SMS
	- Trigger: payment worker publishes `booking_confirmed` events
	- SES used for email; SNS SMS or Twilio for SMS

Monitoring & Resilience
	- CloudWatch alarms:
		- SQS ApproximateNumberOfMessagesVisible > 10,000 -> pager
		- DLQ messages > 0 -> pager
		- Redis cluster node down -> degrade mode alert
	- Circuit breaker for SQS publish failures: after 60s of publish errors, API falls back to synchronous payment path with reduced concurrency (operator-controlled)

Annotations to Part A
	- Redis SETNX locks: see `docs/CONCURRENCY.md` (handles hot 500k burst)
	- Cache keys & TTLs: see `docs/CACHE.md` (availability TTL 30s)
	- Async payments + SQS config: see `docs/QUEUE.md` (visibility timeout 120s, maxReceive 3)
	- Schema design: see `docs/SCHEMA.md` (UUID booking IDs, seats.version optimistic lock)

Data flows (examples)
	- User requests availability page: Browser -> CloudFront (cache hit -> HTML/JSON) ; on cache miss CloudFront -> ALB -> Node -> Read Replica for seatmap/availability or Node reads Redis key availability:{event}:{cat}.
	- User selects seat & clicks Pay: Node -> check holds:{userId} in Redis; attempt seat_lock:{event}:{seat} SETNX -> if acquired, DB UPDATE seats SET status='held', held_by=userId, held_until=now()+10m; create bookings (status=pending); publish SQS message -> respond 202 to user.
	- Payment worker processes SQS: charge gateway -> on success, in DB transaction update bookings.status='confirmed' and seats.status='booked' -> publish SNS -> delete SQS message.

