# Changelog

All notable changes to POS.Go are recorded here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project uses [semantic versioning](https://semver.org/spec/v2.0.0.html)
for the app version, with a separate monotonically increasing `versionCode` for
Google Play. See [`VERSIONING.md`](VERSIONING.md) for the full policy.

Two version lines are tracked and they are **not** the same thing:

- **App versions** (`1.0.0`, `versionCode 3`) — the Android build you install.
- **Platform versions** (`v1.0`, `v1.2`) — the hosted backend and admin console.
  A merchant sees these as "the service was updated"; no app update is required.

---

## 1.0.0 — 2026-09-28

`versionCode 3` · first public release on Google Play · package `com.xamltech.pos_go`

### Offline checkout

- Sales keep working without internet. When the network drops, the till queues
  the sale on the device and replays it on reconnect — no lost tickets, no
  double charges.
- The product catalog is cached on the device, so the grid opens instantly and
  works with no connection at all.
- Device-to-device sync: a second till picks up the same catalog and stock.

### Payments

- Cash, card and mobile money, and **split payments** across any combination of
  them on a single sale.
- Configurable discount policy per store: off, warn, hard cap, or require manager
  approval by PIN above a threshold.
- Tips, VAT and cash rounding, with the rounding delta shown in the payment sheet
  and printed on the receipt.

### Cash sessions

- One cash session per terminal: opening float, then a close with counted cash.
- Z-report showing expected versus counted cash and the difference.
- Session history per device for end-of-day reconciliation.
- Closing the drawer can require a manager PIN.

### Stock and suppliers

- Live stock badges on every product card, with low-stock thresholds and an
  optional block on selling out-of-stock items.
- Manual adjustments for damaged goods, restock and stock counts.
- **Invoice scanning**: photograph a supplier invoice, review the lines detected,
  apply in one tap. Stock and cost update, and unknown items can be created on
  the spot. Purchases history keeps supplier and invoice numbers.
- Pharmacy lot tracking with first-expiry-first-out consumption, and size/colour
  variants for apparel, with per-variant stock and pricing.

### Restaurant mode

- Floors and tables, assign a table to a sale, and split the bill per cover.
- Table and floor names printed on the receipt.

### Refunds and customers

- One-tap refunds, full or partial, by quantity — stock is restored
  automatically and loyalty points are clawed back.
- Customer records with phone, email and loyalty points earned on every sale.

### Reports

- Today: revenue, profit with and without VAT, average sale, items sold.
- Payment mix, top products and a per-cashier breakdown.
- Sales history with a receipt preview.

### Receipts and printers

- Thermal printer pairing over Wi-Fi or by IP, supporting 80 mm and 58 mm paper.
- Receipt preview on the device, with paper size, copies, cut and theme settings.

### Self-order and QR menu

- Publish a QR menu for tables or the counter. Customers scan, order and request
  items; staff approve from the same device.

### Language, roles and Egypt

- Full Arabic and English interface with right-to-left layout.
- Per-user roles (owner, manager, cashier) and per-user access levels, with a
  manager PIN for refunds, over-cap discounts and closing the drawer.
- EGP and Egyptian phone formats out of the box.
- Create a store in the app and start on a free 15-day trial, no card needed.
  The trial store is provisioned instantly with a demo catalog already loaded.

### Privacy and accounts

- Privacy policy published in Arabic and English.
- Account deletion request page with a stated turnaround.

### Fixes in this release

- The login form no longer overflows on narrow phones; the form scrolls and
  dropdowns expand within the available width.
- The IME "Next" key no longer skips the email field in Arabic right-to-left
  layout.
- Product descriptions are returned on every catalog read path, not just the
  detail view.
- Demo and seeded catalogs use valid check-digit EAN-13 barcodes, so barcode
  scanning works against seeded data.

---

## Platform milestones

Backend and admin-console releases. No app update is required to receive these.

### v1.2 — 2026-09-17

- **SaaS admin control panel**: overview with per-business-type user and revenue
  breakdown; tenant list with search and type/status/plan filters, pagination and
  pull-to-refresh; per-tenant analytics drill-down in the tenant's own currency.
- Suspend and re-activate a tenant from the control panel, with an audit trail.
- **Branded public site and privacy policy** served by the API itself, in Arabic
  and English, with no login required. A localized privacy policy link was added
  to the app's sign-in screen.
- Full backend test suite made hermetic, so it passes identically whether or not
  a local `.env` is loaded.

### v1.0 — 2026-09-15

- **Merchant onboarding**: pick one of seven business types (coffee shop,
  restaurant, retail, book store, mobile shop, computer shop, grocery), sign up,
  and land in a working store — owner account, 15-day trial, and a demo catalog of
  ten real products per vertical with EGP prices, images, descriptions and
  check-digit-valid EAN-13 barcodes. Sign-in happens automatically.
- Duplicate email registration is rejected rather than creating a second store.
- Trial expiry is enforced at sign-in, with a clear renewal message.
- **Production hardening**: the API is bound to loopback only, TLS terminates at
  the reverse proxy with a trusted auto-renewing certificate, the firewall
  defaults to deny, and intrusion protection is active. Direct port exposure was
  removed.

### 2026-09-25 — service update

- Backend rebuilt on Go 1.26 with dependency updates.
- Community console added to the web admin.
- Production deploy pipeline given a mandatory post-deploy smoke gate, so a bad
  deploy fails the run instead of being discovered later.

### 2026-09-28 — service update

- Google Search Console verification served from every public surface, so the
  store listing can be indexed.
