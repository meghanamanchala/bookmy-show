Book my show d0wn fall

Constraints
-----------

- **Concurrent users (T+0):** 500,000 users expected to hit "Book Now" at the same instant. Peak RPS depends on client retry behavior, but the system must tolerate a very large burst (hundreds of thousands of requests in seconds).
- **Acceptable double-bookings:** 0 — the system must ensure at-most-one confirmed booking per seat (no exceptions).
- **AWS budget:** $2,000 / month — infrastructure choices must fit this budget (limited number and size of instances, minimal managed services beyond RDS/ElastiCache/SQS).

How these constraints interact
--------------------------------
- Speed vs Correctness: Strong locking guarantees (pessimistic locks) increase latency and reduce throughput; optimistic or external locks (Redis) reduce DB pressure but add operational cost and complexity.
- Scale vs Budget: Horizontal scale (many API servers, DB replicas, large Redis clusters) is expensive — we must design for efficiency (cache heavily, use async payments, minimize DB-held time).
- Correctness vs Budget: The safest approaches (3-node Redis Redlock, large RDS clusters) cost more; the design chooses a hybrid approach (Redis for hot-path seat holds, Postgres for final ACID commit) to balance correctness and cost.

Repository layout and next steps
--------------------------------
See `docs/SCHEMA.md`, `docs/CONCURRENCY.md`, `docs/CACHE.md`, and `docs/QUEUE.md` for the complete Part A deliverables.
