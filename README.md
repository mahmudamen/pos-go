# POS.Go

**Offline-first point of sale for Egyptian shops, cafés and restaurants.**

Arabic-first, EGP-native, built to keep selling when the network does not.

| | |
|---|---|
| Google Play | <https://play.google.com/store/apps/details?id=com.xamltech.pos_go> |
| Package | `com.xamltech.pos_go` |
| Current release | 1.0.0 (versionCode 3) |
| Website | <https://posgo.xamltech.com> |
| Privacy policy | <https://posgo.xamltech.com/private> |
| Delete your data | <https://posgo.xamltech.com/delete-account> |
| Support | support@xamltech.com |

## What this repository is

**Documentation only.** It holds the release notes, the version history, the
technical roadmap and product information for POS.Go.

It deliberately contains **no source code**. The application source (a Go/Gin
backend and a Flutter Android client) is maintained privately; if you want to
report a bug or request a feature, use the channels above.

| Document | What it covers |
|---|---|
| [`CHANGELOG.md`](CHANGELOG.md) | Every shipped version, newest first, with user-facing notes |
| [`ROADMAP.md`](ROADMAP.md) | What is being built next and why |
| [`VERSIONING.md`](VERSIONING.md) | Version numbering, `versionCode` policy, and what counts as a breaking change |
| [`docs/TECHNOLOGY.md`](docs/TECHNOLOGY.md) | How it works: architecture, offline sync, security model, API shape |

## What POS.Go does

**Checkout**

- Product grid with categories, search, barcode scanning and live stock badges
- Split payments across cash, card and mobile money
- Discounts under a configurable policy: off, warn, hard cap, or manager approval by PIN
- Tips, VAT, cash rounding and a printed balance line
- Restaurant mode: assign a table, add a tip, split the bill per cover
- Focus mode for a distraction-free cashier screen

**Cash till**

- One cash session per terminal: opening float, counted closing cash, and a
  Z-report showing expected versus counted
- Session history per device for end-of-day reconciliation

**Stock and suppliers**

- Stock on hand per product, with low-stock warnings and an out-of-stock block
- Manual adjustments for damaged goods, restock and stock counts
- Invoice scanning: photograph a supplier invoice, review the lines detected,
  apply in one tap — stock and cost update, unknown items can be created on the spot
- Purchases history with supplier and invoice numbers

**Reports**

- Today: revenue, profit with and without VAT, average sale, items sold
- Payment mix, top products, per-cashier breakdown
- Sales history with receipt preview and one-tap refunds that restock

**Receipts and printers**

- Thermal printer pairing over Wi-Fi or by IP, 80 mm and 58 mm paper
- Receipt preview on the device, plus paper size, copies, cut and theme settings

**Customers and loyalty**

- Customer records with phone, email and loyalty points earned on every sale

**Self-order and QR menu**

- Publish a QR menu for tables or the counter: customers scan, order and request
  items; staff approve from the same device

**Offline first**

- Catalog cached on the device, sales queued when the network drops and replayed
  automatically on reconnect
- Device-to-device sync, so a second till sees the same catalog and stock

**Language and roles**

- Full Arabic and English interface with right-to-left layout
- Per-user roles (owner, manager, cashier) and per-user access levels
- Manager PIN for refunds, discounts over the cap and closing the cash drawer

**Built for Egypt**

- EGP and Egyptian phone formats out of the box
- Create your own store in the app and start on a free 15-day trial, no card needed
- VAT-aware profit reporting and an Egyptian community for staff and shop owners

## Getting started

The sign-in screen, the sign-up flow and the privacy pages are public — no
account is needed to look around. To try the full product, tap **Create store**
on the sign-in screen with any email address and password. The trial store is
provisioned instantly, already stocked with a demo catalog, and includes every
feature for 15 days.

## Support

- Email support@xamltech.com
- Self-service: create a store, or read the privacy policy and the
  account-deletion page at the links above

Licensing of the application itself is not stated in this repository; see the
Google Play listing for the terms that apply to the distributed app.
