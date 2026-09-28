# Roadmap

What is being built next, and why. Dates are deliberately absent — the order is
the commitment, the calendar is not.

Anything marked **Shipped** is already in a released version; see
[`CHANGELOG.md`](CHANGELOG.md). The engineering detail behind these items lives
in the private source repository.

---

## Next up — Egyptian tax and accounting (VAT + ETA e-invoicing)

The largest remaining workstream, and the one with a legal clock on it.

Egypt's e-invoicing framework requires:

- **VAT at 14%** under Law 67/2016, with tax-inclusive pricing, per-rate splits,
  discount pro-rating and credit notes on refund.
- **B2B / B2G e-invoices** cleared with the Egyptian Tax Authority (ETA) in real
  time, digitally signed.
- **B2C e-receipts** from registered POS devices, reported within 24–72 hours,
  with a **mandatory printed QR code** on the paper receipt.
- Merchant **TRN** and product **EGS/GS1** codes.

Planned, in order:

1. Tax and VAT schema, plus a VAT engine that handles inclusive prices, per-rate
   splits and discount pro-ration. Every calculation is a pure, unit-tested
   function so figures can be trusted.
2. ETA document builder and e-seal signer for invoices, receipts and credit
   notes, verified byte-for-byte against official samples.
3. Submission outbox and background worker: chunked submission, status polling,
   ETA notification receiver, retry with backoff, and alerting inside the 72-hour
   e-receipt window.
4. Accounting core: chart of accounts, journals and lines with debits equal to
   credits, and a posting engine that flows sales, refunds, register sessions,
   inventory and purchases into the ledger. VAT registers and a monthly VAT
   return worksheet.
5. Merchant onboarding and catalog mapping: TRN and device registration, plus
   EGS/GS1 bulk CSV import with a report of codes that are missing.
6. Reporting: a submission dashboard, VAT register screens, and reconciliation
   between what the till recorded and what ETA accepted.
7. The mandatory ETA QR code added to both the on-screen and printed receipts.
8. Go-live: a pre-production soak, cut-over, triage for rejected and expired
   batches, and an operations runbook.

**Why this is first:** it is the difference between a POS that a shop can run and
one that a shop can file taxes with. Everything else can wait a release.

---

## Next up — POS Copilot community

POS.Go already has staff profiles, companies, and an Egypt-wide member
directory with invitation codes, expertise levels and moderation. Planned next:

- **Knowledge hub**: staff write posts, with categories, drafts versus published,
  featured posts and view counts. Publishing earns expertise points.
- **National job board**: post a role directly, or promote a job already
  advertised in your own store, with Egyptian governorates, EGP salaries and
  applications carrying a resume snapshot.
- **Expertise score and leaderboard**: a ledger of expertise events feeding
  deterministic bronze-to-platinum tiers, with period filters.
- **Strikes and moderation**: weighted strikes that expire, warning/mute/suspend
  thresholds, and a way to lift or revoke one.
- **National badges**: definitions with auto-award rules — first steps, first
  post, popular author, people person, hired, respected, clean record.
- **Tenant badges and endorsements** on staff profiles.
- **Community forum**: threads, posts, comments and reactions, with moderation.

**Why:** retention for the people who use a till all day is worth more than
another checkout feature.

---

## Shipped

Closed out, listed so the shape of the product is legible.

**Checkout gap**

- Receipts as a hardware-agnostic ESC/POS byte stream plus an on-screen preview
- Stock-on-hand badges on the product grid
- Offline sale queue with idempotent replay
- Split payments, and payment methods throughout
- Inventory adjustments with reason codes

**Operations and security**

- Server-enforced discount limits by role — the server never trusts the client
- Manager PIN for refunds, over-cap discounts and closing the drawer, with
  lockout after repeated failures
- Five-tier per-user access levels (none, cashier, advanced, manager, admin) with
  a per-action permission matrix
- Granular `resource.action` permissions enforced server-side
- Cash sessions per terminal with Z-report reconciliation
- Multi-device sync: a change feed for reads and replay-safe commands for writes

**Verticals**

- Restaurant: floors, tables, table assignment, split bills, tips
- Pharmacy: lot and serial tracking, first-expiry-first-out at checkout
- Fashion and textile: size and colour variants with their own stock and price
- Loyalty points on sales, with a points ledger

**Analytics and platform**

- Dashboard with today's figures, payment mix, top products, per-cashier breakdown
- SaaS control panel: tenant list, filters, per-tenant analytics, suspend and
  re-activate, tenant backup export
- Plan limits enforced for users and products
- Client crash and event telemetry, tenant-scoped

**Engineering**

- Row-level security forced on every tenant table
- Redis-backed rate limiting, structured tenant-scoped logs, Prometheus metrics
  behind an allowlist
- OpenAPI specification generated from the live route table, so it cannot drift
- Integration test environment with real Postgres and Redis

---

## Considering, not committed

These have been evaluated and are not scheduled. They may never ship.

| Item | Position |
|---|---|
| **In-app purchase for scan credits** | Google Play Billing becomes mandatory the moment an app can spend real money. Scan credits are currently granted, not sold; if this changes, Billing is a prerequisite, not a detail. |
| **Multi-location / warehouse stock** | Needs a warehouse and transfer model. `available` stock currently mirrors on-hand rather than being computed separately. |
| **Kitchen display systems** | A separate screen, not a phone feature. Restaurant mode is complete without it. |
| **Textile cutting orders** | A genuinely popular feature in comparable deployments, but it needs a production schema that no vertical currently models. |
| **Buy-X-get-Y, coupons, promotion rules** | Loyalty is points-on-sales only. Promotions need a rules engine. |
| **Argon2id password hashing** | Passwords use bcrypt. Swapping affects stored hashes, so it is a migration, not a config change. |
| **Subscription billing payments** | A payment provider is out of scope. Plans, limits and the control panel ship; taking money for them does not. |
