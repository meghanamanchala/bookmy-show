Concurrency strategy analysis for ShowTime (Part A)

Summary (one sentence)
We choose a HYBRID strategy: Redis SETNX locks for the hot seat-hold path at sale time, with PostgreSQL optimistic/transactional confirmation for final booking commits.

Why this hybrid?
- Redis SETNX (lightweight distributed lock) handles very high lock-op throughput during the 12:00:00 burst and keeps DB connections free.
- PostgreSQL remains the source of truth for final ACID commits, schema-level constraints, and optimistic version checks.
- This balances throughput (handle hundreds of thousands of concurrent attempts) with correctness (DB finalization) while staying within the $2,000/mo budget.

Option A — PostgreSQL `SELECT ... FOR UPDATE` (pessimistic row locks)

How it prevents double-booking
- Begin a transaction, `SELECT ... FOR UPDATE` the seat rows filtered by `status='available'`, then `UPDATE` to `held` and `INSERT` booking rows, then `COMMIT`. The DB enforces serial access to those rows until commit.

Capacity/Pool math (realistic example)
- Let `max_connections` = 500 (PgBouncer pool),
- Assume 80% of requests are non-payment reads/holds with avg DB time 20ms (0.02s), and 20% are payment-related operations that would hold a connection for 800ms (0.8s).
- Average connection hold time per request = 0.8 * 0.02 + 0.2 * 0.8 = 0.016 + 0.16 = 0.176 seconds.
- Connections held = RPS × 0.176.
- Max sustainable RPS ≈ 500 / 0.176 ≈ 2,840 RPS.

Conclusion for Option A
- At sustained RPS > ~3k, the DB pool exhausts. A flash-sale with 500k simultaneous users far exceeds this. PostgreSQL-only locking is simple and safe but cannot meet the burst throughput required here within the budget.

Risks and mitigations
- Deadlocks for multi-seat bookings: mitigate by ordering seat IDs consistently when locking, and enforcing a small retry/backoff on serialization failures.

Option B — Redis `SETNX` distributed locks (optimistic TTL locks)

How it prevents double-booking
- Before touching DB, acquire per-seat lock `seat_lock:{event_id}:{seat_id}` using `SET key value NX EX ttl`. If acquire fails, return temporary-unavailable.
- After acquiring, read DB seat status and perform the minimal DB update to mark `held`. Release lock via a Lua script that verifies ownership (compare stored lockValue) before `DEL`.

Failure modes and TTL choice
- If Redis fails mid-lock, the DB remains authoritative; without the lock two workers could both read 'available' and race to update DB — but the DB's version check or unique seat constraints will prevent final double-booking if implemented.
- TTL tradeoff: choose short TTL to reduce stuck-lock risk (e.g., 30s–120s) but long enough to survive expected slow operations. For our async flow we hold seats for up to 10 minutes; Redis lock is only for the short critical section (DB check + update) so TTL = 5s is enough for the lock operation itself. The longer user-held state is represented in DB `held_until`.

Operational considerations
- Requires an ElastiCache Redis node/cluster (cost). Plan accounts for a small cluster (3 x cache.r6g.large) to be resilient and support high ops/sec under budget.

The Hybrid Choice (final)
- Hot-path (user taps "Book Now" and selects seat): acquire Redis lock per seat (key: `seat_lock:{event_id}:{seat_id}`), do a fast DB check-and-mark (UPDATE seats SET status='held', held_by=..., held_until=now()+10min WHERE id=$1 AND status='available' RETURNING version), release Redis lock.
- Payment and finalization: booking is created with status `pending` and published to queue; the payment worker does a DB transaction to move `pending` -> `confirmed` and mark seats `booked`. DB version checks and transactional updates are used at confirmation time.

Why not pure Redis final authority?
- Using Redis as the final source of truth (and atomically persisting bookings only to Redis) risks data loss and complex reconciliation. Keeping PostgreSQL as the authoritative store preserves ACID guarantees and makes audits and refunds straightforward.

When would we switch strategies?
- If budget allowed and traffic patterns showed repeated saturations, move to a more robust distributed lock scheme (Redlock across multiple independent Redis clusters) or introduce a dedicated high-throughput seat allocation service with sharded in-memory state.

Known limitations
- This hybrid design relies on Redis availability for the hot path; Redis failure increases latency and falls back to DB locking (degraded throughput). The DB remains authoritative so correctness is preserved.
