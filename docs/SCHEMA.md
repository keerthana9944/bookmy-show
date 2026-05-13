-- ShowTime (BookMyShow) Schema
-- PostgreSQL DDL for events, venues, seats, users, bookings, booking_seats

-- VENUES
CREATE TABLE venues (
	id          SERIAL PRIMARY KEY,
	name        VARCHAR(255) NOT NULL,
	city        VARCHAR(100) NOT NULL,
	capacity    INT NOT NULL CHECK (capacity > 0),
	created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_venues_city ON venues(city);

-- EVENTS
CREATE TABLE events (
	id            SERIAL PRIMARY KEY,
	name          VARCHAR(255) NOT NULL,
	venue_id      INT NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
	starts_at     TIMESTAMPTZ NOT NULL,
	total_seats   INT NOT NULL CHECK (total_seats > 0),
	status        VARCHAR(20) NOT NULL DEFAULT 'upcoming'
									CHECK (status IN ('upcoming','on_sale','sold_out','cancelled')),
	created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_events_starts ON events(starts_at);
CREATE INDEX idx_events_status ON events(status);

-- USERS
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE TABLE users (
	id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	email         VARCHAR(255) UNIQUE NOT NULL,
	phone         VARCHAR(30) UNIQUE,
	name          VARCHAR(100) NOT NULL,
	created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- SEATS
CREATE TABLE seats (
	id            SERIAL PRIMARY KEY,
	event_id      INT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
	section       VARCHAR(50) NOT NULL,
	row_label     VARCHAR(10) NOT NULL,
	number        INT NOT NULL,
	category      VARCHAR(30) NOT NULL,
	price         DECIMAL(10,2) NOT NULL CHECK (price > 0),
	status        VARCHAR(20) NOT NULL DEFAULT 'available'
									CHECK (status IN ('available','held','booked')),
	held_until    TIMESTAMPTZ,
	held_by       UUID REFERENCES users(id) ON DELETE SET NULL,
	version       INT NOT NULL DEFAULT 0,
	created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_seats_event_pos
	ON seats(event_id, section, row_label, number);
CREATE INDEX idx_seats_event_status ON seats(event_id, status);

-- BOOKINGS
CREATE TABLE bookings (
	id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
	event_id      INT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
	status        VARCHAR(20) NOT NULL DEFAULT 'pending'
									CHECK (status IN ('pending','confirmed','failed','refunded')),
	total_amount  DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
	payment_ref   VARCHAR(200),
	created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
	confirmed_at  TIMESTAMPTZ
);
CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);
CREATE INDEX idx_bookings_event ON bookings(event_id);
CREATE INDEX idx_bookings_status ON bookings(status)
	WHERE status IN ('pending','failed');

-- BOOKING_SEATS (junction table)
CREATE TABLE booking_seats (
	booking_id  UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
	seat_id     INT NOT NULL REFERENCES seats(id) ON DELETE RESTRICT,
	PRIMARY KEY (booking_id, seat_id)
);
CREATE INDEX idx_bs_seat ON booking_seats(seat_id);

-- Helpful view (optional): quick lookup of seat status by event
CREATE VIEW v_event_seat_status AS
SELECT s.id AS seat_id, s.event_id, s.section, s.row_label, s.number, s.category, s.status
FROM seats s;

-- Commentary / Rationale

Why UUID for booking.id instead of SERIAL?
- Security & idempotency: UUIDs are unguessable (avoid enumeration) and can be
	generated client-side to provide idempotent booking creation (same bookingId on retry).

Why does seats have a `version` column?
- Optimistic concurrency control: application can attempt
	UPDATE seats SET ... , version = version + 1 WHERE id = $1 AND version = $2
	and detect conflicts if 0 rows were updated. This avoids heavy locking for confirmations.

Why `held_until` instead of only app-level holds?
- `held_until` persists hold state in the DB so background jobs can reliably release expired holds
	(cron or worker), preventing orphaned holds after crashes. It is the source-of-truth for hold expiry.

Why a partial index on bookings.status?
- The payment worker frequently queries unresolved bookings (pending/failed). A partial index
	reduces index size and improves scan speed for the hot lookup while keeping historical data compact.

