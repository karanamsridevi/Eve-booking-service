# EVE Diagnostics Booking API

Backend service for diagnostic-test bookings with simulated payments and an idempotent payment webhook.
**FastAPI · SQLAlchemy 2 · PostgreSQL · JWT · pytest · Docker**

## Run locally

### Option A: Docker (Postgres included)
```bash
docker compose up --build
# API: http://localhost:8000   Swagger UI: http://localhost:8000/docs
docker compose exec api python -m scripts.seed      # optional demo data
```

### Option B: plain Python (SQLite by default, zero setup)
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # set DATABASE_URL to a Postgres URL if you have one
uvicorn app.main:app --reload
python -m scripts.seed          # optional demo data
```

### Tests
```bash
pytest -q        # 34 tests, in-memory SQLite, no external services needed
```

An admin is any user who signs up with an email listed in `ADMIN_EMAILS` (env var). Admins manage the catalogue; everyone else can only read it.

## API overview

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/auth/signup` | – | Register |
| POST | `/auth/login` | – | Get JWT |
| GET | `/auth/me` | user | Current user |
| GET/POST | `/tests/` | GET public, POST admin | Diagnostic test catalogue |
| GET | `/centres/?location=&test_id=&limit=&offset=` | – | List centres with tests & prices |
| GET | `/centres/{id}` | – | One centre |
| POST | `/centres/` | admin | Create centre |
| PUT | `/centres/{id}/tests` | admin | Offer a test at a centre / change its price |
| POST | `/bookings/` | user | Create booking (status `PENDING`) |
| GET | `/bookings/?status=` | user | My bookings (paginated) |
| GET | `/bookings/{id}` | owner | One booking |
| POST | `/bookings/{id}/cancel` | owner | Cancel (idempotent) |
| POST | `/payments/` | owner | Simulated payment |
| POST | `/payments/webhook/` | HMAC signature | Provider status updates |

### Example flow
```bash
B=http://localhost:8000
curl -s $B/auth/signup -H 'Content-Type: application/json' \
  -d '{"email":"alice@example.com","full_name":"Alice","password":"supersecret1"}'
TOKEN=$(curl -s $B/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"alice@example.com","password":"supersecret1"}' | python -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')

curl -s $B/centres/                                  # find centre_id / test_id
curl -s $B/bookings/ -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"centre_id":1,"test_id":1,"appointment_at":"2026-12-01T10:00:00+05:30"}'
# -> {"id":1,"amount":"499.00","status":"PENDING",...}

curl -s $B/payments/ -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"booking_id":1}'                              # random SUCCESS/FAILED (80% success by default)
```

`POST /payments/` accepts an optional `simulate_outcome` (`SUCCESS` | `FAILED` | `PENDING`) to force a result. `PENDING` leaves the payment open to be settled by the webhook.

### Webhook
```bash
BODY='{"event_id":"evt_123","provider_reference":"pay_abc...","status":"SUCCESS"}'
SIG=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET" | awk '{print $2}')
curl -s $B/payments/webhook/ -H "X-Signature: $SIG" -H 'Content-Type: application/json' -d "$BODY"
# -> {"result":"processed"}   (replay it -> {"result":"duplicate"})
```

## Design

### Schema
```
users(id, email UQ, full_name, hashed_password, is_admin, created_at)
diagnostic_tests(id, name UQ, description)
centres(id, name, location IDX, created_at)            UQ(name, location)
centre_tests(id, centre_id FK, test_id FK, price)      UQ(centre_id, test_id)   -- price per centre
bookings(id, user_id FK, centre_id FK, test_id FK, appointment_at, amount, status, created_at, updated_at)
payments(id, booking_id FK, amount, status, provider_reference UQ, created_at, updated_at)
          └─ partial UNIQUE(booking_id) WHERE status='SUCCESS'
webhook_events(id, event_id UQ, payload JSON, received_at)
```
- **`centre_tests`** is the many-to-many between centres and tests, carrying the centre-specific price.
- **`bookings.amount`** is a snapshot of the price at booking time, so later price changes never rewrite history. The client never sends an amount.
- **`payments`** is separate from bookings: a booking can have several attempts (fail, then retry), but the partial unique index guarantees **at most one SUCCESS per booking** at DB level.
- **`webhook_events`** is the idempotency ledger.
- Money is `NUMERIC(10,2)` (returned as strings in JSON to avoid float errors).

### State machine
```
Booking:  PENDING ──pay SUCCESS──▶ CONFIRMED
          PENDING ──pay FAILED───▶ FAILED ──retry SUCCESS──▶ CONFIRMED
          any (not yet cancelled) ──cancel──▶ CANCELLED
Payment:  PENDING ──▶ SUCCESS | FAILED   (terminal states are final)
```
Both `POST /payments/` and the webhook go through one function (`services/payments.py`), so the rules live in one place. A failed payment never downgrades a CONFIRMED/CANCELLED booking.

### Webhook idempotency & safety
1. **Signature**: HMAC-SHA256 over the raw body (`X-Signature`), constant-time compare → 401 if wrong.
2. **Idempotency key**: `event_id` is inserted into `webhook_events` (UNIQUE) *in the same transaction* as the state change. A replay hits `IntegrityError` → rolled back → `200 {"result":"duplicate"}`. Concurrent duplicates are safe: the DB serialises on the unique key (and the payment row is locked with `SELECT … FOR UPDATE`).
3. **No creation**: the webhook only updates an existing payment (looked up by `provider_reference`); it never creates payments or bookings, so replays cannot duplicate them.
4. **Out-of-order/conflicting events** (e.g. FAILED arriving after SUCCESS) are recorded and ignored, with a warning log.
5. Unknown `provider_reference` → 404 *without* recording the event, so the provider's retry still works later.
6. Always 2xx for already-handled events so the provider stops retrying.

### Edge cases handled
Invalid payloads (422) · past appointment times · centre doesn't offer the test · duplicate active booking for the same slot (409) · duplicate signup (409) · wrong password vs unknown user (same 401) · missing/invalid/expired JWT (401) · non-admin catalogue writes (403) · other users' bookings return 404 (no ID probing) · paying a confirmed/cancelled booking (409) · second payment while one is in flight (409) · retrying after a failed payment · cancelling twice · payment success arriving for an already-cancelled booking (booking stays cancelled, payment recorded, warning logged for refund) · bad webhook signature/payload · repeated webhook events.

## Assumptions
- Single currency, price per centre per test; one booking = one test.
- No slot capacity/availability model: any future time is bookable (duplicates by the same user are blocked).
- Naive datetimes are treated as UTC; stored as UTC.
- Cancelling a CONFIRMED booking does not trigger a (simulated) refund.
- Tables are created at startup via `create_all` for simplicity.
- Admin role is derived from `ADMIN_EMAILS` at signup; no admin-management UI.

## What I'd improve with more time
- Alembic migrations instead of `create_all`.
- Refund flow and payment-expiry job (Celery) that fails PENDING payments after N minutes.
- Refresh tokens / token revocation; login rate limiting (e.g. Redis + slowapi).
- Appointment slots with capacity and operating hours, and a unique constraint to stop double-booking a slot.
- Webhook: timestamp in the signature to prevent replay of old signed requests; dead-letter/retry queue.
- Redis cache for the read-heavy centre catalogue.
- Request-ID middleware and JSON logs; CI pipeline running tests and linting.
- Concurrency tests against real Postgres (current suite uses SQLite, where row locks are no-ops).
