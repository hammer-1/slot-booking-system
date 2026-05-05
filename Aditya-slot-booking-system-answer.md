# Aditya singh - Backend / Full-Stack Aptitude Test – Answers

A quick note before I start: I’ve written SQL/pseudo-code helps I’ve put it in,
otherwise I just wrote out the reasoning. I’m assuming Postgres + Node
backend with Redis, since that’s what most marketplace stacks look like.

---

## Problem 1 – Slot-Based Booking System (30 Marks)

**Question:** Design a slot-based booking system where a partner can accept only one
booking per time slot. Prevent double booking, keep booking creation atomic, and make it
concurrency-safe.

### How I’m thinking about it

The core problem is a race condition. Two users hit “Book 4–5 PM with Partner X” at the
same millisecond. Both reads see the slot as free, both inserts succeed, and now the
partner has two bookings for the same hour. So whatever I design has to make “check + insert”
behave as one atomic step from the database’s point of view.

There are basically three honest ways to solve this:

1. A **unique constraint** on `(partner_id, slot_start)` – the DB itself rejects the second
   insert. This is my favourite because it doesn’t rely on the application being careful.
2. A **`SELECT ... FOR UPDATE`** on the slot row inside a transaction – good when slots are
   pre-generated rows.
3. A **distributed lock** (Redis/Redlock) – I’d only reach for this if the booking write
   touches multiple databases. For a single Postgres, it’s overkill.

I’ll go with option 1 as the primary defence and option 2 as a backup when slots are
pre-generated (which is more realistic for a marketplace – partners define their
availability ahead of time).

### Schema

```sql
-- Slots a partner makes available. Pre-generated when partner sets their schedule.
CREATE TABLE partner_slots (
    id              BIGSERIAL PRIMARY KEY,
    partner_id      BIGINT NOT NULL REFERENCES partners(id),
    slot_start      TIMESTAMPTZ NOT NULL,
    slot_end        TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'available'
                    CHECK (status IN ('available','held','booked','blocked')),
    version         INT NOT NULL DEFAULT 0,         -- for optimistic locking if needed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- This is the key line. One slot per partner per start time, period.
    CONSTRAINT uq_partner_slot UNIQUE (partner_id, slot_start)
);

CREATE INDEX idx_slots_partner_status_time
    ON partner_slots (partner_id, status, slot_start);

CREATE TABLE bookings (
    id              BIGSERIAL PRIMARY KEY,
    customer_id     BIGINT NOT NULL REFERENCES customers(id),
    partner_id      BIGINT NOT NULL REFERENCES partners(id),
    slot_id         BIGINT NOT NULL REFERENCES partner_slots(id),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','confirmed','cancelled','completed')),
    amount          NUMERIC(10,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Belt-and-braces: even if slot logic fails, you cannot have two active
    -- bookings on the same slot.
    CONSTRAINT uq_active_slot
        EXCLUDE (slot_id WITH =) WHERE (status IN ('pending','confirmed'))
);
```

Two layers of safety: the unique constraint on the slot itself, plus a partial exclusion
constraint on the booking that ignores cancelled bookings (so a re-book is allowed after a
cancellation).

### Booking flow (pseudo-code)

```
function createBooking(customerId, partnerId, slotId, amount):
    BEGIN TRANSACTION

    -- Lock the row so other concurrent attempts queue up behind us.
    slot = SELECT * FROM partner_slots
           WHERE id = slotId AND partner_id = partnerId
           FOR UPDATE

    if slot is null:
        ROLLBACK; throw NotFound("slot")

    if slot.status != 'available':
        ROLLBACK; throw Conflict("slot already taken")

    UPDATE partner_slots
       SET status = 'booked', updated_at = now(), version = version + 1
     WHERE id = slotId

    try:
        booking = INSERT INTO bookings (customer_id, partner_id, slot_id, amount, status)
                  VALUES (customerId, partnerId, slotId, amount, 'pending')
                  RETURNING *
    catch UniqueViolation:
        -- exclusion constraint fired, second writer lost the race
        ROLLBACK; throw Conflict("slot already taken")

    COMMIT
    return booking
```

### Why this is concurrency-safe

- `SELECT ... FOR UPDATE` serialises any two requests that target the same slot row. The
  second one waits, then sees `status = 'booked'` and bails out cleanly.
- If somehow a request bypasses that path (different code path, retry, etc.), the
  `EXCLUDE` constraint on `bookings` is the last line of defence.
- The whole thing runs in one transaction, so either the slot flips to `booked` AND the
  booking row exists, or neither happens.

### Things I’d add in a real product

- A short-lived **hold** state (5–10 min) so the user has time to pay before the slot is
  released. A background job (or `held_until` column + cron) reverts unpaid holds.
- Idempotency key from the client so a retried POST doesn’t create a second booking.
- For very hot partners, push the “check availability” read to Redis but always confirm
  the write against Postgres.

---

## Problem 2 – Partner Assignment Logic (20 Marks)

**Question:** Assign a partner based on availability, lowest active workload, and same
city. Return the best `partner_id` and explain tie-breaking.

### How I’m thinking about it

This is a ranked filter. Hard filters first (must be in the same city, must be available
at that time), then a soft ranking (least busy wins). Ties happen often in practice –
imagine two partners both with 2 active jobs in Pune. I need a deterministic rule so the
same input always returns the same partner, otherwise debugging becomes painful.

My tie-break order:

1. Lower active workload (count of `pending` + `confirmed` bookings right now).
2. Higher rating – a small quality nudge.
3. Most recently active – rewards partners who actually use the platform.
4. Lowest `partner_id` – purely deterministic fallback so the result is stable.

I’d compute this in SQL because doing it in app code means pulling N rows just to sort
them, which gets ugly fast.

### SQL

```sql
SELECT p.id AS partner_id
FROM partners p
JOIN partner_slots s
       ON s.partner_id = p.id
      AND s.slot_start = :requested_slot_start
      AND s.status     = 'available'
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS active_jobs
    FROM bookings b
    WHERE b.partner_id = p.id
      AND b.status IN ('pending','confirmed')
) wl ON TRUE
WHERE p.city_id      = :city_id
  AND p.is_active    = TRUE
ORDER BY wl.active_jobs ASC,
         p.rating       DESC,
         p.last_active_at DESC,
         p.id            ASC
LIMIT 1;
```

### Pseudo-code wrapper

```
function assignPartner(cityId, slotStart):
    row = db.queryOne(SQL_ABOVE, {cityId, slotStart})
    if row is null:
        return null            -- caller decides whether to widen city or notify ops
    return row.partner_id
```

---

## Problem 3 – Payment Webhook Handling (25 Marks)

**Question:** Design a payments table and webhook handling that is idempotent, handles
duplicate events, and safely updates booking and payment status.

### How I’m thinking about it

Payment gateways (Razorpay, Stripe, etc.) retry webhooks. They will send the same event
two, three, sometimes ten times. They’ll also occasionally send events out of order. So
the handler has to be:

- **Idempotent** – processing the same event twice = same final state, no double credits.
- **Order-tolerant** – a `payment.captured` arriving before `payment.authorized` shouldn’t
  break things.
- **Verifiable** – signature check before anything else.
- **Auditable** – I want to see every raw event we ever received.

The trick that solves most of this is storing the gateway’s event ID with a unique
constraint. If the row already exists, we know we’ve seen it.

### Schema

```sql
CREATE TABLE payments (
    id                  BIGSERIAL PRIMARY KEY,
    booking_id          BIGINT NOT NULL REFERENCES bookings(id),
    gateway             TEXT NOT NULL,                  -- 'razorpay', 'stripe'...
    gateway_payment_id  TEXT NOT NULL,                  -- pay_XXXX
    amount              NUMERIC(10,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'INR',
    status              TEXT NOT NULL                   -- created/authorized/captured/failed/refunded
                        CHECK (status IN ('created','authorized','captured','failed','refunded','partially_refunded')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (gateway, gateway_payment_id)
);

-- Every webhook ever received. The unique key here is what makes us idempotent.
CREATE TABLE payment_webhook_events (
    id              BIGSERIAL PRIMARY KEY,
    gateway         TEXT NOT NULL,
    event_id        TEXT NOT NULL,                      -- gateway's unique event id
    event_type      TEXT NOT NULL,                      -- payment.captured, refund.processed, ...
    payment_id      BIGINT REFERENCES payments(id),
    raw_payload     JSONB NOT NULL,
    signature       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'received'    -- received/processed/failed/skipped
                    CHECK (status IN ('received','processed','failed','skipped')),
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ,
    error           TEXT,
    UNIQUE (gateway, event_id)
);

CREATE INDEX idx_webhook_status ON payment_webhook_events (status, received_at);
```

### Handler (pseudo-code)

```
POST /webhooks/payments/:gateway
function handleWebhook(req):
    rawBody = req.rawBody
    signature = req.headers['x-signature']

    -- 1. Verify signature first. If this fails, drop on the floor.
    if not verifySignature(rawBody, signature, SECRET[gateway]):
        return 401

    payload = JSON.parse(rawBody)
    eventId = payload.id

    -- 2. Persist the raw event. Unique constraint kills duplicates here.
    try:
        event = INSERT INTO payment_webhook_events
                (gateway, event_id, event_type, raw_payload, signature)
                VALUES (gateway, eventId, payload.event, payload, signature)
                RETURNING *
    catch UniqueViolation:
        -- We've already seen this event. Tell the gateway "yep, got it"
        -- so it stops retrying. Do nothing else.
        return 200

    -- 3. Process inside a transaction. If anything throws, mark the event failed
    --    and return 500 so the gateway retries later.
    try:
        BEGIN
        processEvent(event)
        UPDATE payment_webhook_events
           SET status = 'processed', processed_at = now()
         WHERE id = event.id
        COMMIT
        return 200
    catch e:
        ROLLBACK
        UPDATE payment_webhook_events
           SET status = 'failed', error = e.message
         WHERE id = event.id
        return 500   -- gateway will retry
```

```
function processEvent(event):
    payload = event.raw_payload
    -- Lock the payment row so two webhooks for the same payment don't race.
    payment = SELECT * FROM payments
              WHERE gateway = event.gateway
                AND gateway_payment_id = payload.payment.id
              FOR UPDATE

    if payment is null:
        -- Out-of-order: capture arrived before we created the payment row.
        payment = INSERT INTO payments (booking_id, gateway, gateway_payment_id,
                                        amount, currency, status)
                  VALUES (payload.booking_id, event.gateway, payload.payment.id,
                          payload.amount, payload.currency, 'created')
                  RETURNING *

    -- State-machine guard. Don't move backwards.
    newStatus = mapEventToStatus(event.event_type)        -- e.g. captured
    if not isValidTransition(payment.status, newStatus):
        -- e.g. we already marked it 'refunded', ignore an old 'captured'
        return

    UPDATE payments SET status = newStatus, updated_at = now()
     WHERE id = payment.id

    if newStatus == 'captured':
        UPDATE bookings SET status = 'confirmed' WHERE id = payment.booking_id
    elif newStatus == 'failed':
        -- Release the slot so someone else can book it.
        UPDATE bookings SET status = 'cancelled' WHERE id = payment.booking_id
        UPDATE partner_slots SET status = 'available'
         WHERE id = (SELECT slot_id FROM bookings WHERE id = payment.booking_id)
```

### Why this is safe

- **Duplicate events:** `UNIQUE (gateway, event_id)` makes the second insert fail – we
  short-circuit and return 200.
- **Atomic updates:** payment + booking + slot all change inside one transaction. If
  anything fails, the gateway retries and we try again from a clean state.
- **Out-of-order events:** the state machine refuses backwards transitions
  (`refunded -> captured` is silently ignored).
- **Concurrent webhooks for the same payment:** `SELECT ... FOR UPDATE` serialises them.
- **Signature first:** anything unsigned never even hits the database.

---

## Problem 4 – Cancellation & Refund Flow (15 Marks)

**Question:** Cancel bookings with full or partial refunds based on assignment status.
Keep money consistent using transactions.

### How I’m thinking about it

The refund amount depends on **what state the booking is in when it’s cancelled**:

- Booking still `pending` (not paid yet) → nothing to refund, just mark cancelled.
- Paid but partner not yet assigned → full refund. Customer hasn’t consumed anything.
- Paid + partner assigned, but cancelled well before slot start → full refund minus a
  small platform fee (configurable).
- Paid + assigned + cancelled close to slot start (say < 2h) → partial refund (e.g. 50%).
  Partner blocked their calendar, they deserve compensation.
- Slot already started or completed → no refund.

The dangerous part is making sure: money state, booking state, and slot state all change
together, and the call to the payment gateway either happens or we know it didn’t.

The safe pattern is to do the **DB writes first** in one transaction, mark a refund as
`pending`, then call the gateway, then update the refund row to `processed` (or `failed`)
based on the response. If the process crashes between, a worker picks up `pending` refunds
and retries. This is the standard “outbox-ish” approach.

### Schema additions

```sql
CREATE TABLE refunds (
    id                  BIGSERIAL PRIMARY KEY,
    payment_id          BIGINT NOT NULL REFERENCES payments(id),
    booking_id          BIGINT NOT NULL REFERENCES bookings(id),
    amount              NUMERIC(10,2) NOT NULL,
    reason              TEXT,
    status              TEXT NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','processed','failed')),
    gateway_refund_id   TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ
);
```

### Cancellation flow

```
function cancelBooking(bookingId, actorId, reason):
    BEGIN TRANSACTION

    booking = SELECT * FROM bookings WHERE id = bookingId FOR UPDATE
    if booking.status in ('cancelled','completed'):
        ROLLBACK; throw Conflict("already finalised")

    slot = SELECT * FROM partner_slots WHERE id = booking.slot_id FOR UPDATE
    payment = SELECT * FROM payments
              WHERE booking_id = bookingId AND status = 'captured'
              FOR UPDATE                                  -- may be null

    refundAmount = computeRefund(booking, slot, payment)

    UPDATE bookings
       SET status = 'cancelled', cancelled_at = now(), cancel_reason = reason
     WHERE id = bookingId

    UPDATE partner_slots
       SET status = 'available'
     WHERE id = slot.id

    refund = null
    if payment is not null and refundAmount > 0:
        refund = INSERT INTO refunds (payment_id, booking_id, amount, reason, status)
                 VALUES (payment.id, bookingId, refundAmount, reason, 'pending')
                 RETURNING *

    COMMIT

    -- Outside the txn: hit the gateway. If this fails the row is still 'pending'
    -- and a retry worker will pick it up.
    if refund is not null:
        try:
            resp = gateway.refund(payment.gateway_payment_id, refundAmount)
            UPDATE refunds
               SET status = 'processed',
                   gateway_refund_id = resp.id,
                   processed_at = now()
             WHERE id = refund.id

            if refundAmount == payment.amount:
                UPDATE payments SET status = 'refunded' WHERE id = payment.id
            else:
                UPDATE payments SET status = 'partially_refunded' WHERE id = payment.id
        catch e:
            UPDATE refunds SET status = 'failed' WHERE id = refund.id
            -- worker will retry; alert ops
```

```
function computeRefund(booking, slot, payment):
    if payment is null: return 0                          -- never paid

    hoursToSlot = (slot.slot_start - now()) / 1h

    if booking.partner_id is null:
        return payment.amount                             -- not assigned, full refund
    if hoursToSlot >= 24:
        return payment.amount - PLATFORM_FEE              -- full minus small fee
    if hoursToSlot >= 2:
        return payment.amount * 0.5                       -- partial
    return 0                                              -- too late
```

### Why money stays consistent

- All DB state changes (booking, slot, refund row) happen in one transaction. There is no
  in-between state where “booking is cancelled but slot is still booked.”
- The gateway call is _outside_ the transaction on purpose – I never want to hold a DB
  lock while waiting on a third-party API.
- The refund row is the source of truth. If the gateway call fails or the server crashes,
  the refund stays `pending` and a worker retries it. We never silently lose a refund.
- Refund amount is computed from server-side rules, not trusted from the client.

---

## Problem 5 – Performance Optimization (10 Marks)

**Question:** Indexing, caching strategy, and cache invalidation for the partner
availability API.

### Indexing

The hot query is roughly: “give me available partners in city X for slot Y.” So the
indexes I’d actually create:

```sql
-- For the slot lookup itself.
CREATE INDEX idx_slots_lookup
    ON partner_slots (partner_id, slot_start, status);

-- For the city + active filter on partners.
CREATE INDEX idx_partners_city_active
    ON partners (city_id, is_active);

-- For the workload subquery in the assignment SQL.
CREATE INDEX idx_bookings_partner_status
    ON bookings (partner_id, status);

-- Webhook idempotency lookup (already covered by the unique constraint).
-- Refund retry worker:
CREATE INDEX idx_refunds_pending
    ON refunds (status) WHERE status = 'pending';
```

A couple of things I’d resist doing: indexing low-cardinality columns alone (like
`status` by itself) and adding a separate index per column. Composite indexes that mirror
the actual `WHERE` + `ORDER BY` of the hot queries do far more work.

### Caching strategy

Availability is read constantly and changes only when someone books or a partner edits
their schedule – classic cache candidate.

- **Layer:** Redis, keyed by `availability:{city_id}:{date}`. Value is a small JSON list
  of `{partner_id, slot_start}` for the day.
- **TTL:** short, maybe 60 seconds. Even with perfect invalidation, a TTL is a safety net
  against bugs leaving stale data forever.
- **Read path:** check Redis → on miss, query Postgres → write back to Redis.
- **Don’t cache the booking write path.** Writes always go to Postgres so the unique
  constraint stays authoritative.

For _very_ hot single-partner reads (think a celebrity stylist), I’d also cache
`partner:{id}:slots:{date}` individually so invalidating one partner doesn’t blow away the
whole city.

### Cache invalidation

This is the part people get wrong. Two rules I stick to:

1. **Invalidate on write, not on read.** Whenever a booking is created/cancelled or a
   partner edits their slots, delete the relevant cache keys _inside the same code path_
   right after the DB transaction commits.
2. **Invalidate the smallest blast radius.** If partner 42 in Pune books a slot, drop
   `partner:42:slots:2026-05-06` and `availability:pune:2026-05-06` – not the whole city
   for a year.

```
afterBookingCommit(booking):
    redis.del(f"partner:{booking.partner_id}:slots:{booking.slot_date}")
    redis.del(f"availability:{booking.city_id}:{booking.slot_date}")
```

If the system grows to multiple app servers and ordering matters, I’d switch to a
pub/sub-based invalidation (publish a “slot changed” event, every node drops its local
cache). For a single Redis it’s not needed.

### Other things I’d watch

- **Connection pooling** – PgBouncer in front of Postgres. The booking endpoint is short
  and bursty, exactly the workload that benefits from pooling.
- **Read replicas** for analytics / admin dashboards so they never compete with booking
  writes.
- **Rate limit the availability API** per IP / per user; it’s the easiest endpoint to
  abuse for scraping.
- **Monitor slow queries** with `pg_stat_statements`. Indexes drift in usefulness as
  data grows; I’d revisit them every quarter rather than setting and forgetting.
