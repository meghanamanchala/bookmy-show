## Decision: Concurrency strategy (Redis SETNX vs PostgreSQL FOR UPDATE)

**Context:** A single seat must be protected from double-booking during a 500k-user flash sale while staying within a $2,000/month AWS budget.

**Options considered:**
1. PostgreSQL `SELECT ... FOR UPDATE` (pessimistic row locking) - simple, no extra infra, DB enforces serial access. Fails at very high burst RPS due to DB connection pool exhaustion.
2. Redis `SETNX` distributed locks - sub-ms lock acquisition, offloads contention from DB, requires Redis cluster. More complex and adds cost.
3. Hybrid (chosen) - Redis SETNX for hot-path seat holds; PostgreSQL transactions + optimistic locking for final confirmation.

**Why chosen:** Redis handles the hot lock throughput at 12:00:00 and keeps DB connections free for short transactions. PostgreSQL remains authoritative for final ACID commits, preventing data loss and simplifying refunds/audit. The hybrid approach fits the budget (small Redis cluster + RDS) while scaling to the required burst.

**Tradeoffs accepted:** Requires Redis availability; a full Redis outage degrades throughput and requires fallback behavior. The design accepts operational complexity (lock ownership, TTL tuning) to meet throughput.

**Revision trigger:** If sustained sales or other workloads push beyond Redis ops/sec of our cluster or budget increases, consider Redlock across independent Redis clusters or a dedicated seat allocation microservice.

---

## Decision: Cache invalidation approach (Event-driven targeted invalidation vs TTL-only)

**Context:** Seat availability counts are extremely hot-read during sales and must be as fresh as possible.

**Options considered:**
1. TTL-only (e.g., 30s) - simple but allows stale counts up to TTL.
2. Event-driven targeted invalidation (chosen) - on seat state changes delete specific availability key; TTL as safety net.

**Why chosen:** Event-driven invalidation ensures availability counts update immediately after a seat changes state, preventing misleading "2 seats left" UX. TTL-only leaves unacceptable stale windows during a sale.

**Tradeoffs accepted:** More Redis write operations on invalidation; requires careful targeting to avoid cache stampede.

**Revision trigger:** If Redis write throughput becomes a bottleneck, consider batched invalidation or approximate counters with streams.

---

## Decision: Primary key for bookings - UUID vs SERIAL

**Context:** Need idempotency for retries and to avoid predictable, enumerable booking IDs.

**Options considered:**
1. SERIAL (auto-increment integer) - compact, performant, but predictable and not client-friendly for idempotency.
2. UUID (chosen) - unguessable, client-generatable for idempotency, integrates well with distributed systems.

**Why chosen:** UUID allows clients to pre-generate booking IDs and retry safely (INSERT ... ON CONFLICT DO NOTHING). It prevents enumeration attacks and supports globally unique IDs across services.

**Tradeoffs accepted:** Slightly larger index sizes and storage overhead versus SERIAL.

**Revision trigger:** If index size / performance becomes problematic at extreme scale, migrate to KSUID or Snowflake-style compact IDs.

---

## Decision: SQS visibility timeout = 120s and MaxReceiveCount = 3

**Context:** Payment processing (gateway calls + DB commits + notifications) may take variable time; we must avoid double-processing while allowing retries for transient failures.

**Options considered:**
1. Short visibility (30s) - lower worker hold but risk of duplicate processing if payment or DB commit is slow.
2. Long visibility (120s) (chosen) - gives workers ample time to process messages, reduces duplicate attempts; MaxReceiveCount=3 routes persistent failures to DLQ.

**Why chosen:** 120s ~ 2× expected worst-case payment+DB commit latency, balancing between duplicate processing risk and timely retries. MaxReceiveCount=3 provides room for transient gateway retries while ensuring poison messages are surfaced to DLQ.

**Tradeoffs accepted:** Longer visibility holds messages longer, which can increase time-to-retry in case of stuck workers; operational cost of DLQ handling.

**Revision trigger:** If payment gateway latency patterns change dramatically, visibility should be tuned accordingly.

