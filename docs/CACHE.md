
Cache design for ShowTime (Part A)

Principles
- Cache counts and static shapes, never individual per-seat availability as the source of truth.
- Use cache-aside (delete on write) for targeted invalidation; TTL acts as a safety net.

Entries and keys

1) Event details (static-ish)
- Key: `event:{event_id}`
- Value: JSON: { id, name, venue_id, start_time, status, metadata }
- TTL: 3600s (1 hour). Reason: event metadata changes rarely; a 1-hour TTL reduces DB reads and is easy to invalidate on event update.
- Invalidate: on `events` UPDATE/INSERT/DELETE → delete `event:{event_id}`.

2) Seat availability count per event per category
- Key: `availability:{event_id}:{category}`
- Value: integer count (available seats in that category)
- TTL: 30s. Reason: UI tolerates short staleness (30s) but this is the highest-read item during sale. 30s balances cache thundering risk vs user expectations.
- Invalidate: on any seat status change for that event+category (held/booked/released) → `DEL availability:{event_id}:{category}`. This targeted invalidation keeps counts fresh.

3) Seat map layout (static structure)
- Key: `seatmap:{event_id}`
- Value: JSON layout (sections, rows, seat numbers, categories)
- TTL: 86400s (24h). Reason: layout rarely changes; keep long TTL and only invalidate on event cancellation or layout change.

What NOT to cache
- Do NOT cache individual seat status (e.g., `seat:123:status`). Reason: caching this would create stale view windows leading to race conditions; locks and DB are the source of truth.

Invalidation strategy (cache-aside, targeted invalidation)

When a seat status changes (hold, book, release):

1. Update DB (transactional source of truth).
2. Targeted invalidation: `DEL availability:{event_id}:{category}` (and optionally `DEL event:{event_id}` only if event metadata changed).
3. Do NOT flush global cache; only delete affected keys.

Pseudocode: cache-aside read pattern for availability

async function getAvailabilityCount(eventId, category) {
	const cacheKey = `availability:${eventId}:${category}`;
	const cached = await redis.get(cacheKey);
	if (cached !== null) return parseInt(cached, 10);

	const result = await db.query(
		'SELECT COUNT(*) FROM seats WHERE event_id = $1 AND category = $2 AND status = $3',
		[eventId, category, 'available']
	);
	const count = parseInt(result.rows[0].count, 10);
	await redis.setex(cacheKey, 30, String(count));
	return count;
}

Seat update pseudocode (in app server)

async function updateSeatStatus(seatId, eventId, category, newStatus) {
	const client = await db.connect();
	try {
		await client.query('BEGIN');
		await client.query('UPDATE seats SET status = $1 WHERE id = $2', [newStatus, seatId]);
		await redis.del(`availability:${eventId}:${category}`);
		await client.query('COMMIT');
	} catch (err) {
		await client.query('ROLLBACK');
		throw err;
	} finally {
		client.release();
	}
}

TTL vs event-driven invalidation
- We use targeted event-driven invalidation on writes (`DEL`) because during a flash sale having up-to-date availability counts is critical. TTL is a fallback to recover from missed invalidations.
