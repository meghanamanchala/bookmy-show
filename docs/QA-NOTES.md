# QA Notes — Panel Questions & Answers

Q1: What happens if Redis crashes during a seat hold?
- Answer: Three layers: (1) 3-node Redis cluster for HA; (2) if full cluster outage prevents lock acquisition, API returns 503 instead of proceeding unprotected; (3) DB optimistic version checks/unique constraints remain authoritative — concurrent DB updates will not create duplicate confirmed bookings. Net: Redis outage degrades throughput but does not allow silent double-bookings.

Q2: Replica lag (500ms) shows stale availability — what happens when user books?
- Answer: UI reads (replica/cache) may be stale but booking operations always verify on primary under a lock. If seat already booked, booking fails with 'seat already taken'. Replica lag is a UX issue only, documented in the UI.

Q3: How to prevent a user holding 200 seats across tabs?
- Answer: Add per-user hold counter in Redis (`holds:{userId}`) with TTL 15m and a configurable max (e.g., 8). Reject further holds (429) when limit reached. Also reduce hold TTL during active sale (10m -> 3m) and require session-token tied payment.

Q4: SQS outage — API returns 202 but nothing processed. How detect & recover?
- Answer: CloudWatch alarms on SQS publish/drop and DLQ; status polling endpoint `GET /bookings/:id` shows `pending` so client can surface message; after 60s of publish errors circuit breaker enables synchronous fallback at reduced throughput. Messages are durable in SQS and processed when workers recover.

Q5: Budget blowout during peak - how to control costs?
- Answer: Treat $2,000 as steady-state; peaks are amortized across event revenue. Controls: use spot instances for workers, aggressive CloudFront caching, tune autoscaling cooldown to scale in quickly, and pre-purchase capacity for known events if needed.

Self-assessment notes
- Felt strong on Redis failure and replica lag answers. Q3 (abuse) required introducing per-user limit (added in DESIGN-UPDATES.md). Q4 (SQS fallback) required explicit circuit breaker and polling endpoint (also added).
