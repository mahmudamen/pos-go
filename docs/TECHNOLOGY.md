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
                         │                ├──►  Redis 7   (rate limits)
Self-order  ──┘          │                └──►  Tesseract (invoice OCR)
   QR page                │
                          └──►  OpenAPI document generated from the live routes
```

- **Client** — Flutter, Android. Plain `http`, `sqflite` for the offline queue,
  secure storage for credentials. No Firebase, no proprietary sync service: the
  app talks to the POS API and nothing else.
- **Server** — Go with Gin. Single binary. JSON over HTTPS, everything under
  `/v1`, with a consistent `{"data": ...}` / `{"error": ...}` envelope.
- **Database** — PostgreSQL 16, migrations applied forward-only by `goose` at
  deploy time. Redis for rate limiting only; the API stays up if Redis is down.
- **OCR** — a local Tesseract process, in the API container. No third-party OCR
  service, so no supplier invoice leaves the deployment.
- **Admin** — a React single-page app for the SaaS control panel, served as static
  files.

Everything is containerised. Base images are pinned by digest, the API runs as a
non-root user, and the API process is bound to loopback with TLS terminating at
the reverse proxy.

## Go backend

Go was chosen for a specific reason: a point of sale cannot afford to be
unavailable, and a single static binary with a small memory footprint is far
easier to reason about operationally than a managed application server.

- **Single binary, few moving parts.** The API is one process with a connection
  pool. There is no application server to tune and no application-level
  clustering to debug.
- **Goroutines and context cancellation map onto the domain.** A sale is a
  transaction; everything that touches the database takes a `context.Context`, so
  a cancelled client request actually releases its database work instead of
  holding a connection open.
- **Interfaces at the edges, not everywhere.** Where a boundary genuinely needs
  swapping — the OCR engine, the rate limiter — there is a small interface with
  two implementations. The rest is concrete code, because an interface with one
  implementation is a cost with no benefit.
- **Pure functions for anything that must be right.** Payment splitting, discount
  policy, refund line allocation, first-expiry-first-out lot ordering and invite
  code generation are pure functions with no database and no I/O. That is what
  makes them exhaustively unit-testable, and they are where the business rules
  live.
- **The race detector runs in CI.** `go test -race` is part of the gate, not an
  occasional check.
- **Transactions own the invariants.** Stock is locked and decremented in the
  same transaction that writes the sale, and a uniqueness violation is caught
  inside a savepoint so the transaction stays usable. Rules that involve money
  are decided inside the transaction, never in the client.

The result is that a shop owner is served by a process that starts instantly,
holds little memory, and fails in ways that are easy to diagnose.

## Flutter client

The app targets Android and is written in Flutter, with a few deliberate
constraints:

- **No Firebase, no analytics SDK, no advertising SDK.** The app's only network
  dependency is the POS API. This is not minimalism for its own sake — it is the
  same decision that makes offline operation possible, and it means the app talks
  to nobody else.
- **`sqflite` is the offline queue, not a cache of convenience.** A sale is
  written to local SQLite *before* the cashier is told it succeeded, so a crash
  or a dead battery cannot lose a completed sale.
- **Secure storage for credentials.** Session tokens and remembered logins live
  in the platform keystore via `flutter_secure_storage`, not in shared
  preferences.
- **Arabic is the default, English is the alternative.** The interface is
  Arabic-first with a full right-to-left layout, and the choice persists across
  restarts. Right-to-left is a real source of bugs — the "Next" key on the
  keyboard walks fields in a reading order that does not match visual order — so
  the login form drives focus explicitly and is covered by widget tests.
- **Money is formatted through one helper.** One place converts minor units to a
  display string, so no screen can invent its own rounding.
- **Rendering is kept in local state.** A POS screen is rebuilt often; the
  pattern is deliberately plain so a stall is visible rather than hidden behind a
  framework.

## Invoice OCR

Photographing a supplier invoice and turning it into stock is a real feature, not
a placeholder. The pipeline is:

1. The device photographs or picks the invoice and uploads it to the API.
2. The API checks a per-tenant usage allowance **before** spending any engine
   time, so a store cannot burn scans it has no allowance for.
3. Tesseract runs locally over the image with Arabic and English language data,
   invoked as a subprocess — no CGO, so the main binary stays portable and only
   the runtime container needs the OCR packages.
4. The recognised text is parsed into candidate line items: product name,
   quantity, unit and line total. Units are normalised, so a supplier writing
   "pcs", "piece" or "قطعة" resolves to the same thing.
5. Each candidate line is matched against the store's catalog by normalised name,
   with a confidence score. A line that does not match well enough is **not**
   silently attached to a product — it comes back for review.
6. The device shows a review screen. The user corrects anything wrong, and can
   create a product that the invoice mentions but the store does not sell.
7. Applying the invoice updates stock and cost, and is recorded in purchases
   history with the supplier and invoice number.

The review step is the point. OCR on a creased, badly lit, handwritten invoice is
a suggestion, never an authority, so a human confirms before stock moves.

The engine is configuration-gated. If OCR is disabled or the binary is missing,
the scan endpoint answers cleanly and the rest of the purchasing feature — the
invoice list, applying an invoice — keeps working.

### Metering

Scans are metered per tenant on rolling day, week and month windows. A store that
exhausts its windows and its credit balance gets a clear, specific error and a
notification rather than a silent failure; a manager can top up the balance.

Note for anyone considering selling those credits: the top-up currently credits an
internal balance and **does not take payment**. Selling credits for real money
inside the app would make Google Play Billing mandatory, and that is a product
decision, not an implementation detail.

## Database

PostgreSQL, and specifically PostgreSQL's row-level security rather than a
hand-rolled tenant column filter on every query.

- **Isolation is enforced by the database.** Row-level security is *forced* on
  every tenant table, including for the table owner, so a forgotten `WHERE` clause
  cannot leak one store's sales into another's. A request without tenant context
  set sees nothing. Integration tests run against PostgreSQL as a
  **non-superuser**, because a superuser bypasses these policies and would make
  the tests pass while proving nothing.
- **Migrations are forward-only and transactional**, applied by `goose` inside the
  deploy. There is no down-migration path in production; a bad migration is
  rolled forward. This is deliberate — a reverse migration that has been tested is
  safer than an untested one.
- **New tables must be granted explicitly.** The application connects with a
  least-privilege role that is not the table owner, so a new table without a grant
  fails in production while working in development. Grant scripts are part of the
  deploy, and this has caught real bugs.
- **Money is `BIGINT` minor units**, never `FLOAT` and never a decimal type that
  could be read as approximate.
- **Enums are `CHECK` constraints** on `TEXT`, so adding a value is a migration
  rather than a type rewrite.
- **A monotonic `change_seq` per tenant** orders every syncable change, backed by
  triggers. Incremental sync reads it directly instead of diffing tables.
- **Soft deletes for catalog rows**, transmitted to other devices as explicit
  deletions so a second till stops selling a deleted product rather than keeping
  it forever.
- **Observability data is excluded from the change feed.** Client crash reports
  are tenant-scoped and row-level secured, but they are not business state and do
  not belong in a device's sync stream.

A handful of tables are deliberately not row-level secured: the Egypt-wide
community directory and platform administration, which are cross-tenant by
nature and gated in the handler instead. Every other table is secured.

## Money and time

Money is never a float. Every amount is an integer in minor units — piastres
for EGP — named `*_minor` throughout the API. A sale of 16.00 EGP is `1600`. This
removes an entire class of rounding bug from the ledger and the VAT work that
comes next.

## Multi-tenancy

One deployment serves many independent shops. A shop is a *tenant*, and the
tenant boundary is the most important line in the system:

- A tenant owns its catalog, sales, customers, staff, settings, cash sessions and
  history. Nothing is shared between tenants, not even a product definition or a
  barcode — two shops can both sell "Coca-Cola 500ml" as entirely separate
  products with separate stock and separate prices.
- A tenant is created with a business type, and the type selects the vertical it
  can use: a restaurant gets floors and tables, a pharmacy gets lot and expiry
  tracking, an apparel shop gets size and colour variants. The type is not
  cosmetic; it decides which features apply.
- Users belong to exactly one tenant and carry a role and an access level within
  it. There is no cross-tenant user, which is deliberate: it keeps a compromised
  staff account from reaching another shop.
- Localization is per tenant — country, currency, default language — so a store
  formats money and dates the way its market expects.

Isolation is enforced in the database rather than in application code. See
[Database](#database) for the mechanism, and [SaaS controls](#saas-controls) for
the operator's side of the boundary.

## SaaS controls

POS.Go runs as a multi-tenant SaaS. A shop owner sees none of this; it exists so
that an operator can onboard, understand, support and if necessary stop a store
without touching a database by hand.

**Tenant lifecycle.** Stores are created through a public sign-up that provisions
a trial, a business type and a seeded demo catalog in one request. A suspended
store is refused at sign-in with a specific reason rather than a generic failure,
so an owner knows what happened. A stopped store is refused the same way, its
subscriptions are cancelled, and the action is written to an audit log. Every
lifecycle action — activated, suspended, stopped — is audited, and an operator can
re-activate a store afterwards.

**Cross-tenant analytics.** Overview figures aggregate every store. Because
row-level security is forced, the aggregation cannot be a single `SELECT` over
all tenants; it runs in one transaction that iterates tenants and sets the tenant
context for each. Analytics drill into one store — user and product counts, the
day's figures, a revenue trend, top products, recent sales — and are computed in
the tenant's own currency so the numbers mean something to that store's owner.

**Plan limits.** A plan carries a maximum number of users and products, where
zero means unlimited. Limits are checked *inside* the tenant transaction, so the
count that is compared is the count that store can actually see. A store that
hits a limit gets a specific error, not a generic failure.

**Tenant data export.** An operator can stream a complete export of one store's
data as a single JSON document, produced in a transaction with the tenant context
set, so the export is exactly what that store can see and nothing more. Platform
tables are excluded. This is a support tool — recovering a store's data without
shell access to production.

**Separation of duties.** Platform endpoints require a distinct operator role. An
ordinary shop owner, manager or cashier cannot reach them, and the check is on
the role rather than on a hidden flag in the payload. The operator surface is a
separate application, served separately, from the till.

**Trial handling.** Trials are enforced at sign-in. A store past its trial end is
refused with a clear renewal message, and an operator can extend it. Registration
enforces email uniqueness across stores, so one person cannot accidentally create
a second store and lose track of the first.

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
- **No third-party AI service.** Invoice OCR runs as a local Tesseract process
  inside the deployment. No supplier invoice is sent to an external API, and
  there is no API key, per-request cost, or data-processing agreement to manage.
- **No Firebase.** Sync is purpose-built for a till, not a document store.
- **Money as integers, not decimals.** See above.
- **Server-side authority for every rule that costs money.** Discounts, refunds
  and stock are decided in the transaction, not in the app.
- **A human confirms the OCR.** See [Invoice OCR](#invoice-ocr). Recognition is a
  suggestion; stock does not move until someone accepts it.

## What is not here yet

Stated plainly, so nobody has to guess:

- **There is no language model in POS.Go.** No GPT, no local model, no
  inference server, no model fine-tuning.
- **There are no embeddings and no vector store.** Nothing in the system embeds
  text or stores vectors, and there is no similarity search, semantic search or
  RAG — retrieval-augmented generation over shop data does not exist in the
  product.
- **The only machine learning in the product is OCR**: local Tesseract, Arabic
  and English, producing text that a deterministic parser turns into invoice
  lines, plus deterministic name matching against the catalog.

If AI assistance is added later — for example answering questions over a shop's
own sales and stock, or a support assistant inside the control panel — it will be
described here when it ships, and the licensing terms in `LICENSE` already
reserve XAMLtech's rights in the product's material.
