## Update 1: Per-User Seat Hold Limit (Redis counter)

**Triggered by:** Panel Question 3 — seat holding abuse (a single user holding many seats across tabs).

**What changed:**
- Added Redis key `holds:{userId}` (integer counter with TTL 15 minutes).
- API checks `holds:{userId}` before attempting any `seat_lock:*` acquisition. If >= `MAX_HOLDS` (configurable, default 8) return `429 Too Many Requests`.
- Increment `holds:{userId}` on successful lock acquisition; decrement on hold release, booking confirmation, or when background job expires holds.

**Why this is necessary:**
- Prevents a single user from holding disproportionate seats and degrading availability for genuine buyers during hot sales.

**What it costs:**
- One additional Redis GET per hold request and one INCR/DECR on success (+~0.1ms latency). Negligible relative to lock acquisition.

**What it doesn't solve:**
- Multi-account attackers can bypass per-user limits; addressing that requires product-level controls (KYC, payment pre-auth) beyond the architecture.

---

## Update 2: SQS Circuit Breaker & Synchronous Fallback

**Triggered by:** Panel Question 4 — SQS outage during sale.

**What changed:**
- Added a circuit breaker in API servers monitoring SQS publish success rate. If publishes fail for > 60s, API enters degraded mode and falls back to synchronous payment processing (limited concurrency) configurable via feature flag.
- Added booking status polling endpoint `GET /bookings/:id` for clients to poll `pending` state.
- Added CloudWatch alarms for SQS publish failures and DLQ population to trigger on-call.

**Why this is necessary:**
- Avoids silent acceptance of bookings that will never be processed, gives a clear recovery path, and allows clients to present accurate state to users.

**What it costs:**
- Complexity: added circuit-breaker code path and smaller capacity for synchronous fallback. Potentially increased API latency during fallback.

**What it doesn't solve:**
- If payment gateway is also down during fallback, the system cannot process payments; it will signal an outage to users and require manual intervention.

---

## Update 3: Temporary Hold TTL Reduction During Active Sale

**Triggered by:** Panel concerns about long 10-minute holds impacting availability.

**What changed:**
- For active sale windows (configurable event flag), reduce `held_until` duration from 10 minutes to 3 minutes. The seat lock TTL in Redis remains small (5s) and covers only the atomic check-and-mark.

**Why this is necessary:**
- Shorter hold windows free seats faster for other buyers during extremely high contention events.

**What it costs:**
- Slightly higher friction for users who take longer to complete payment; mitigated by keeping UX tight and using fast payment flows.

**What it doesn't solve:**
- Users with slow networks or long payment flows may fail to complete purchase; product decisions (one-click, saved payments) help here.
