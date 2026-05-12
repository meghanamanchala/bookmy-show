-- docs/SCHEMA.md

PostgreSQL schema for ShowTime (BookMyShow competitor)

-- Requires: the `uuid-ossp` extension for uuid_generate_v4()
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Venues
CREATE TABLE venues (
	id BIGSERIAL PRIMARY KEY,
	name TEXT NOT NULL,
	city TEXT NOT NULL,
	capacity INTEGER NOT NULL CHECK (capacity > 0),
	created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now()
);

CREATE INDEX idx_venues_city ON venues(city);

-- Events
CREATE TABLE events (
	id BIGSERIAL PRIMARY KEY,
	venue_id BIGINT NOT NULL REFERENCES venues(id) ON DELETE CASCADE,
	name TEXT NOT NULL,
	start_time TIMESTAMP WITH TIME ZONE NOT NULL,
	status TEXT NOT NULL CHECK (status IN ('upcoming','on_sale','sold_out','cancelled')),
	total_seat_count INTEGER NOT NULL CHECK (total_seat_count >= 0),
	created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now()
);

CREATE INDEX idx_events_start_time ON events(start_time);

-- Users
CREATE TABLE users (
	id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
	email TEXT NOT NULL UNIQUE,
	phone TEXT NOT NULL UNIQUE,
	name TEXT NOT NULL,
	created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now()
);

-- Seats (source-of-truth per seat for an event)
CREATE TABLE seats (
	id BIGSERIAL PRIMARY KEY,
	event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
	section TEXT NOT NULL,
	row TEXT NOT NULL,
	number TEXT NOT NULL,
	price NUMERIC(10,2) NOT NULL CHECK (price > 0),
	category TEXT NOT NULL CHECK (category IN ('VIP','Premium','General')),
	status TEXT NOT NULL CHECK (status IN ('available','held','booked')),
	held_until TIMESTAMP WITH TIME ZONE,
	held_by UUID REFERENCES users(id) ON DELETE SET NULL,
	version INTEGER NOT NULL DEFAULT 0,
	created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
	UNIQUE (event_id, section, row, number)
);

-- Workhorse index used by availability queries and seat updates
CREATE INDEX idx_seats_event_status ON seats(event_id, status);

-- Bookings (orders)
CREATE TABLE bookings (
	id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
	user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
	event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE RESTRICT,
	status TEXT NOT NULL CHECK (status IN ('pending','confirmed','failed','refunded')),
	total_amount NUMERIC(12,2) NOT NULL CHECK (total_amount >= 0),
	payment_reference TEXT,
	created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
	updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now()
);

CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);

-- Partial index: unresolved (pending or failed) bookings are queried frequently by payment workers
CREATE INDEX idx_bookings_unresolved ON bookings(id) WHERE status IN ('pending','failed');

-- Junction table: booking -> seats
CREATE TABLE booking_seats (
	id BIGSERIAL PRIMARY KEY,
	booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
	seat_id BIGINT NOT NULL REFERENCES seats(id) ON DELETE RESTRICT,
	price_paid NUMERIC(10,2) NOT NULL CHECK (price_paid > 0),
	created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
	UNIQUE (seat_id, booking_id)
);

-- Additional useful indexes
CREATE INDEX idx_booking_seats_seat ON booking_seats(seat_id);

-- Commentary / Design Rationale

Why UUID for `bookings.id` and `users.id`?
- UUID v4 makes booking identifiers unguessable and supports client-generated IDs for idempotency. Clients can generate a booking UUID before calling the API; the server can `INSERT ... ON CONFLICT DO NOTHING` to handle retries.

Why `seats.version` (optimistic locking)?
- `version` enables optimistic concurrency control for seat updates: read the row with its version, attempt `UPDATE seats SET version = version + 1 WHERE id = $1 AND version = $read_version`. If zero rows are updated, the caller knows a concurrent modification happened.

Why `held_until` on seats (instead of holding only in app memory)?
- Storing `held_until` in DB ensures holds survive process crashes and lets background jobs release expired holds reliably. It is the authoritative source for when a seat becomes available again.

Why a partial index on `bookings.status`?
- Most bookings will be historical (`confirmed`, `refunded`). Payment workers repeatedly query unresolved bookings (`pending` / `failed`). A partial index covering only unresolved rows keeps that index tiny and extremely fast for the hot worker queries.

Transactional guarantees for multi-seat bookings
- Booking creation + seat status changes must be performed in a transaction when transitioning to `confirmed` or `failed`. For example, on payment success: within a transaction update `bookings.status='confirmed'`, update `seats.status='booked'` for all seat_ids in the booking and increment `version`. If any step fails, rollback to avoid orphaned holds.

Release cron
- A background worker periodically runs: `UPDATE seats SET status='available', held_by=NULL, held_until=NULL WHERE status='held' AND held_until < now()` to release abandoned holds (safety belt in case of crashes).
