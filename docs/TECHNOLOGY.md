# Technology

How POS.Go is built. Written for a technical reader evaluating the product — a
developer, an integrator, or a curious shop owner.

The application source is private. This document describes the shape of the
system, not its internals.

## Shape

```
Android app  ──┐
               ├──►  HTTPS /v1  ──►  Go API  ──►  PostgreSQL 16
Second till ──┘         │                │            (row-level security)
                         │                └──►  Redis 7   (rate limits)
Self-order  ──┘
   QR page
```

- **Client** — Flutter, Android. Plain `http`, `sqflite` for the offline queue,
  secure storage for credentials. No Firebase, no proprietary sync service: the
  app talks to the POS API and nothing else.
- **Server** — Go with Gin. Single binary. JSON over HTTPS, everything under
  `/v1`, with a consistent `{"data": ...}` / `{"error": ...}` envelope.
- **Database** — PostgreSQL 16, migrations applied forward-only by `goose` at
  deploy time. Redis for rate limiting only; the API stays up if Redis is down.
- **Admin** — a React single-page app for the SaaS control panel, served as static
  files.

Everything is containerised. Base images are pinned by digest, the API runs as a
non-root user, and the API process is bound to loopback with TLS terminating at
the reverse proxy.

## Money and time

Money is never a float. Every amount is an integer in minor units — piastres
for EGP — named `*_minor` throughout the API. A sale of 16.00 EGP is `1600`. This
removes an entire class of rounding bug from the ledger and the VAT work that
comes next.

## Multi-tenancy

Every merchant is a tenant. Isolation is enforced in the database, not in
application code:

- Row-level security is **forced** on every tenant table — including for the table
  owner, so a bug in a query cannot leak across stores.
- The API sets the tenant context per transaction; a request without it sees
  nothing.
- A handful of genuinely cross-tenant tables — the Egypt-wide community directory
  and platform administration — are deliberately excluded from row-level security
  and gated in the handler instead. That exception is a documented decision, not
  an oversight.

There is also a platform-level role, held only by the SaaS operators, for
cross-store analytics, tenant lifecycle and backup export.

## Offline first

This is the part that is not standard, so it is worth being precise about it.

**Reads.** The catalog is cached on the device. When a till needs changes, it asks
for a change feed from a cursor it already holds.

**Writes.** A sale taken while offline is written to a local queue in SQLite
*before* the app reports success to the cashier. On reconnect the queued commands
are pushed to a single endpoint that applies them in one transaction.

**Exactly-once is enforced server-side, not hoped for:**

- Every queued command carries a client-generated idempotency key.
- A command is recorded, then applied. A retry with the same key and the same
  payload replays the stored outcome without re-applying — the response says so
  explicitly.
- The same key with a *different* payload is a conflict, not a silent overwrite.
- Business failures such as insufficient stock come back as a conflict, not a
  silent success, so the cashier is told.

The result: a sale cannot be lost, cannot be charged twice, and cannot be
reported as accepted when the server rejected it. Stock is checked and decremented
inside the same transaction that writes the sale, with the row locked.

## Sync between devices

A monotonic change sequence per tenant orders every change. A device pulls from
its cursor, receives a bounded page plus the next cursor, and repeats. Soft-deleted
catalog rows are transmitted as deletions rather than being omitted, so a device
does not keep selling a product another till deleted. A cursor older than the
retained window is rejected explicitly rather than silently skipping changes.

User accounts, sessions and sync command history are deliberately excluded from
the feed.

## Security model

- Passwords hashed with bcrypt at cost 12.
- Short-lived access tokens and longer-lived refresh tokens, kept separate in
  type. Refresh rotates; presenting an already-used refresh token revokes the
  whole family, so a stolen token is usable at most once.
- Sign-in is rate limited per IP, backed by Redis.
- Permission checks are a static `resource.action` matrix evaluated server-side:
  `pos.discount`, `pos.refund`, `inventory.adjust`, `catalog.write` and so on.
  Five per-user access levels supply defaults, and an individual user can be
  narrowed or widened.
- Discount policy is evaluated on the server against the store's configured mode
  — off, warn, cap, or block above a threshold unless a manager authorises with a
  PIN. The client is told what the policy decided and re-checks it, but it is
  never trusted: a modified client gets the same answer as an honest one.
- Manager PINs are bcrypt-hashed, rate limited with a lockout window after
  repeated failures, and never returned by the API.
- Security headers, request size limits, panic recovery and request IDs are
  middleware, applied uniformly.
- Metrics are served on a separate listener behind an IP allowlist, and can be
  switched off entirely.
- Client crash reports carry a tenant and user scope, and are stored
  tenant-isolated.

## Receipts

Receipts are produced as a **hardware-agnostic ESC/POS byte stream** — 32-column
thermal layout, QR code, cut command — rather than being targeted at one printer
model. Any Epson or Star thermal printer accepts the bytes as they are. The same
sale also returns a structured JSON receipt, which is what the on-device preview
renders. The byte layout is covered by byte-level tests, so it cannot regress
silently.

## API surface

- JSON over HTTPS under `/v1`, with a shared data and error envelope.
- Cursor-based change feed and command push for sync, as described above.
- Catalog, sales, refunds, registers, customers, loyalty, inventory, restaurant
  floors and tables, lots, variants, dashboards, tenant settings, and the
  community surface.
- An OpenAPI 3 document is generated from the **live route table** — the same
  table the binary serves from — so the specification cannot drift from what is
  actually running. Write-body schemas are hand-augmented as the contract grows.
- Liveness and readiness probes are separate: readiness reports not-ready when a
  dependency is missing, liveness stays healthy so an orchestrator does not kill a
  process that is merely waiting.

## Testing

- Go: unit tests for the pure logic that has to be right — payment splitting,
  discount policy, VAT-free money maths, refund line allocation, FEFO lot
  ordering, invite codes — plus handler tests, and integration suites against a
  real PostgreSQL running as a **non-superuser**, so the row-level security
  policies are actually exercised rather than bypassed.
- Idempotency and concurrency are tested the way they fail in production: five
  concurrent requests with the same key must produce exactly one sale and one
  stock decrement.
- Flutter: unit tests for the same pure logic mirrored client-side, and widget
  tests that drive real flows — offline checkout into the queue and back out via
  sync push, the manager-PIN dialog, stock badge tiers, receipt layout.
- Integration tests that need a device or a database skip cleanly when neither is
  present, so the suite is green offline as well as in CI.
- `go test -race` and static analysis are part of the gate, alongside a security
  scanner.

## Deliberate non-choices

- **No third-party analytics or advertising SDK.** The app talks to the POS API
  and nothing else, which is also why it can work offline.
- **No Firebase.** Sync is purpose-built for a till, not a document store.
- **Money as integers, not decimals.** See above.
- **Server-side authority for every rule that costs money.** Discounts, refunds
  and stock are decided in the transaction, not in the app.
