# Roadmap

What is being built next, and why. Dates are deliberately absent — the order is
the commitment, the calendar is not.

Anything marked **Shipped** is already in a released version; see
[`CHANGELOG.md`](CHANGELOG.md). The engineering detail behind these items lives
in the private source repository.

This roadmap is informed by watching what comparable Egyptian merchant platforms
actually ship — where they earn their keep, and where their documentation gives
away that a feature is unfinished. Where a decision here exists to do something
*differently*, the reasoning is written down rather than assumed.

## At a glance

| | Workstream | Why it is where it is |
|---|---|---|
| **1** | [Egyptian VAT + ETA e-invoicing](#1-egyptian-vat--eta-e-invoicing) | A legal clock. A shop cannot file taxes without it. |
| **2** | [POS Copilot community](#2-pos-copilot-community) | Retention for the people who use a till all day. |
| **3** | [Release hardening](#3-release-hardening--before-the-next-update) | The first release reached shops; the next one is where rough edges show up. |
| — | [AI assistance over shop data](#considering--ai-assistance-over-shop-data) | Deliberately **not started** — a product decision, not a queued ticket. |
| — | [Evaluated, not committed](#considering-not-committed) | In-app purchase, warehouses, KDS, cutting orders, promotions, Argon2id. |

**Workstream 1** runs in eight ordered stages: tax schema and VAT engine → ETA
document builder and e-seal signer → submission outbox and worker → accounting
core and general ledger → merchant onboarding and catalog mapping → reporting and
reconciliation → receipt QR → go-live soak and cut-over.

The first three are load-bearing. A VAT engine that cannot be trusted about
tax-inclusive prices and discount pro-ration is worse than no VAT engine, and a
document the tax authority rejects in real time is worse than no submission at
all.

**Workstream 3** before the next update: on-device re-test of the release build
(sign-in, a sale, a printed receipt, an offline sale replayed), thermal printer
pairing across the models a shop is likely to own on both 80 mm and 58 mm paper,
the self-order QR menu driven end to end as a customer rather than as staff, and
invoice scanning tuned against crumpled, badly lit, handwritten supplier
invoices — the case where the review step earns its keep.

If you are looking for something specific and it is not here, it has either
shipped or been declined on purpose. Both are documented rather than dropped.

---

## 1. Egyptian VAT + ETA e-invoicing

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

## Considering — AI assistance over shop data

POS.Go has **no language model, no embeddings and no vector store today.** The
only machine learning in the product is local OCR for supplier invoices. See
[`docs/TECHNOLOGY.md`](docs/TECHNOLOGY.md#what-is-not-here-yet).

If it is added, the likely shape is retrieval-augmented assistance over data the
shop already owns, rather than a chatbot bolted onto a till:

- **Ask your own data** — "what sold best last month", "which products are about
  to run out", "why is this branch's margin down". Answered from the shop's own
  sales, stock and purchase history, with the numbers shown rather than a bare
  claim.
- **Invoice and product assistance** — suggest a category, unit or existing
  product for a poorly recognised invoice line, and help reconcile a supplier
  statement.
- **Control-panel support assistant** — answer an operator's questions about a
  tenant's usage and state.

Constraints that would apply, and are not negotiable by accident:

- **Tenant data must not leak between stores.** Any retrieval index would have to
  be tenant-scoped as strictly as the database is, and the forced row-level
  security that protects PostgreSQL would have to be matched rather than
  replaced.
- **Offline stays offline.** A till that loses the network must keep selling. AI
  assistance is a connected-device convenience and must never sit in the path of
  a sale.
- **Nothing is sent to a third party without the shop's explicit agreement**, and
  the OCR precedent — local processing inside the deployment — is the default
  that a hosted option would have to beat.
- **Answers must be checkable.** A figure an assistant produces has to be
  traceable to the query that produced it, because a shop owner cannot audit a
  confident wrong number.

Status: **not started.** This is a genuine product decision, not a queued ticket.

---

## 2. POS Copilot community

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

### Drawn from adjacent products

The community work is not invented in a vacuum. Three things come from watching
what comparable Egyptian merchant platforms actually ship, and one thing comes
from what they get wrong:

- **Referral by invitation, not by banner.** The national directory already
  issues single-use invitation codes, and that is the whole mechanic a referral
  programme needs. What is new is rewarding the inviter: a share link that
  credits **scan credits**, not cash, which keeps the reward inside the product
  and keeps payout, tax and fraud handling out of scope.
- **Payouts the way Egyptian shops actually get paid.** If a community
  programme ever does pay out, bank transfer and mobile wallet are the rails
  people use. That is a deliberate deferral, not an oversight — a till that has
  to withhold tax from an individual's commission is a different product.
- **A place to talk that is not the app.** Merchant communities in this market
  live on Telegram. A channel for releases and support costs very little and
  reaches the people who will not install a community tab.
- **Where they are weak: automation.** Comparable platforms document their
  referral withdrawal as a manual, support-contacted process. Anything POS.Go
  builds here should be automatic and self-service from the start, because that
  is the cheapest possible way to be better than the alternative.

**One hard constraint, inherited from the app store.** Google Play Billing
becomes mandatory the moment an app can spend real money. Rewarding referrals
with **granted** scan credits stays outside that rule; the moment credits become
**purchasable**, the whole surface has to move to Play Billing, mobile wallets
have to be reconciled, and this item stops being a small feature. That decision
belongs before the first line of referral code is written, not after.

---

## 3. Release hardening — before the next update

The first release reached shops. The next one is where the rough edges get
found, so this is deliberately unglamorous work with an obvious finish line:

- **On-device re-test of the release build** — sign-in, a sale, a printed
  receipt, and an offline sale replayed after reconnecting. The release build is
  not the same binary as the debug build, and the difference has to be proven on
  a real device rather than assumed.
- **Thermal printer pairing** across the printer models a shop is likely to
  already own, on both 80 mm and 58 mm paper. Receipt bytes are already correct
  and hardware-agnostic; what remains is pairing and paper handling.
- **Self-order QR menu, driven as a customer** — scan to order to staff approval
  to payment, end to end, rather than only the staff-side approval screen.
- **Invoice scanning against bad input** — crumpled, shadowed, handwritten and
  skewed supplier invoices. The review step already exists; this is about making
  the suggestion good enough that correcting it is quick.
- **Offline behaviour under real network failure**, not just a clean disconnect:
  a captive portal, a dead SIM, a router that has not finished booting.

**Why first:** every one of these is a support call that otherwise arrives on
the day a shop tries to use the product, and none of them need a design decision.

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
