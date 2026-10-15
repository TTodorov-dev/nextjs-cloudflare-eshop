# Build Specification: Laser Engraving Shop (Phase A)

Version 2, 2026-10-06. Supersedes the v1 project brief. Items marked **[ACC]** need confirmation from the accountant, **[LAW]** from a lawyer, before launch. Section 15 lists them all.

---

## 1. Product

A website for a small laser engraving shop in Bulgaria selling engraved products (wood, MDF, glass, leather, metal and similar).

Goals:

1. A working online shop that takes real orders.
2. A portfolio piece for the developer. All code, including the admin panel, is written in-house. Code quality, architecture, tests, CI/CD and documentation must be presentable to employers.

### Phases

- **Phase A (this document):** standard, non-personalized products, card payment via Stripe.
- **Phase A+ (right after launch):** cash on delivery through a postal money transfer (ППП) contract.
- **Phase B:** personalization (text or logo, live preview).
- **Phase C:** custom jobs (upload, quote, pay).

Phase A must not block A+, B or C. Section 8 lists the fields that keep the model extensible.

### Constraints

- Solo developer.
- Hosting budget about 5 USD per month.
- No product photos, texts or brand assets yet. The site must look complete with placeholder content.

---

## 2. Stack

| Topic      | Decision                                                                                                      | Notes                                                                                                                          |
| ---------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Framework  | Next.js 16 (App Router), TypeScript strict, exact versions pinned                                             | Pin the newest Next minor that the current `@opennextjs/cloudflare` release notes list as supported. Verified in milestone A1. |
| Hosting    | Cloudflare Workers Paid (5 USD/month) via `@opennextjs/cloudflare`                                            | Bundle limit 10 MiB compressed. CI checks size with `wrangler deploy --dry-run`.                                               |
| Next cache | OpenNext R2 incremental cache + D1 tag cache                                                                  | D1 tag cache is fine at low traffic. Switch to the Durable Object tag cache if traffic grows.                                  |
| Database   | Neon Postgres (AWS eu-central-1, Frankfurt) + Hyperdrive + Drizzle ORM on `pg`                                | Two Hyperdrive configs, see 2.1.                                                                                               |
| Files      | Cloudflare R2. Public bucket `media` on `media.<domain>`. Private bucket `private` for documents and backups. | S3 API via `aws4fetch` for presigned uploads.                                                                                  |
| Images     | Cloudflare Images transformations via a custom `next/image` loader                                            | No sharp on Workers.                                                                                                           |
| Auth       | Better Auth: email + password, TOTP 2FA (mandatory), sign-up disabled, single admin seeded from env           | Rate limiting stored in the database, see 11.                                                                                  |
| Admin      | Custom panel at `/admin`                                                                                      | Portfolio requirement.                                                                                                         |
| Payments   | Stripe Checkout, card only (`payment_method_types: ['card']`; Apple Pay and Google Pay work through card)     | Pinned `apiVersion`. `constructEventAsync` for webhooks.                                                                       |
| Email      | Resend (EU region) + React Email                                                                              |                                                                                                                                |
| PDF        | `pdf-lib` + `@pdf-lib/fontkit` with an embedded, subset Cyrillic font                                         | Not `@react-pdf/renderer` (bundle size, memory).                                                                               |
| i18n       | next-intl. Locales at launch: **`bg` (default, unprefixed URLs), `en`**                                       | Schema supports any number of locales. Adding de, nl, ru, pt, pl, ro later is content work only.                               |
| Currency   | **EUR only.** Integer cents. **No BGN amounts anywhere** (storefront, emails, invoices, documents).           | The mandatory dual display period ended on 8 Aug 2026.                                                                         |
| UI         | Tailwind CSS + shadcn/ui. Fonts via `next/font` (self-hosted), Cyrillic + Latin Extended subsets              |                                                                                                                                |
| Analytics  | Cloudflare Web Analytics (cookieless)                                                                         | No cookie banner needed, see 12.                                                                                               |
| Local dev  | Docker Compose: Postgres + Stripe CLI. Storage uses a real R2 dev bucket.                                     | No S3 emulator (MinIO stopped publishing community images in 2025).                                                            |
| CI/CD      | GitHub Actions                                                                                                | See 13.                                                                                                                        |

### 2.1 Hyperdrive bindings

Hyperdrive caches read queries by default (60 s) and does not invalidate them on writes.

- `HYPERDRIVE_CATALOG`: caching **on**. Used only for public catalog reads (products, categories, pages, gallery).
- `HYPERDRIVE`: caching **off** (`--caching-disabled true`). Used for checkout, stock, orders, webhooks, admin, auth, cron jobs.

Database clients are created **per request** and closed after it. No module-level pool. The Better Auth instance is also built per request.

### 2.2 Portability rule

All platform-specific code lives in `src/lib/`: `db.ts`, `storage.ts`, `images.ts`, `stripe.ts`, `email.ts`, `auth.ts`, `env.ts`, `cron.ts`. `env.ts` validates every environment variable with Zod. The rest of the app imports only from these modules. Moving to Node/Docker later touches only these files, the OpenNext cache config and the deploy configuration.

### 2.3 Middleware and auth guard

- `proxy.ts` (or `middleware.ts`, whichever OpenNext supports reliably at the pinned version) handles **only** next-intl locale routing.
- `/admin` is protected by `requireAdmin()` in the admin root layout **and** at the top of every Server Action and route handler under admin. Authorization never depends on middleware.

### 2.4 Scheduled jobs

GitHub Actions cron calls `POST /api/cron/<job>` with a secret bearer token. (Alternative: a custom OpenNext worker entry exporting `scheduled`. Decide in A1; GitHub Actions is the default because it is simpler.)

| Job                          | Schedule     | Purpose                                                                                                 |
| ---------------------------- | ------------ | ------------------------------------------------------------------------------------------------------- |
| `release-stale-reservations` | every 10 min | Release stock for pending orders older than 35 min whose Stripe session is expired or was never created |
| `sync-carrier-offices`       | daily 03:00  | Econt and Speedy office and locker lists                                                                |
| `backup`                     | daily 02:00  | `pg_dump` (runs in the GitHub Action itself) uploaded to the `private` bucket                           |
| `audit-file-reminder`        | monthly, 5th | Email the admin to generate and submit the Н-18 audit file                                              |

---

## 3. Shipping

- **Country:** Bulgaria only. **Carriers:** Econt and Speedy. The model supports more countries, carriers and zones later.
- **Delivery types:** `office`, `locker` (Econtomat / Speedy APT), `address`.
- **Price:** admin settings, VAT included:
  - `shipping_rate_cents` (default 300). Optional per-type override (`shipping_rate_address_cents`).
  - `free_shipping_threshold_cents` (default 5000).
- **Free shipping rule:** the cart's **gross subtotal after discounts** is `>= free_shipping_threshold_cents`. Exactly 50.00 EUR qualifies. A later partial return never triggers a shipping charge.

### Checkout delivery step (our own page, before Stripe)

Collects:

- full name, phone (validated as a Bulgarian number with `libphonenumber-js`, stored in E.164), email
- carrier, delivery type
- **office / locker:** chosen from a searchable list (synced daily). If the list for that carrier is empty or older than 7 days, the field falls back to free text automatically.
- **address:** city (from the carrier's city list, fills the postcode), postcode, street or ж.к./квартал, number, блок, вход, етаж, апартамент (optional), note for the courier (optional)

The chosen office or address is **snapshotted** onto the order (`order_addresses`). Later changes to `carrier_offices` never change an order.

### Stripe

Stripe Checkout runs without address collection. One server-computed shipping option (0 or N cents).

### Labels and tracking

Labels are created manually in the carriers' systems. The admin enters the tracking number, which sends the "shipped" email with the carrier's tracking link.

---

## 4. Stock

- Stock is per variant: `stock` (physical) and `reserved`. Available = `stock - reserved`.
- Customers see only "In stock" / "Out of stock". Unavailable variants cannot be added to the cart.
- **Reservation:** one transaction, variants locked in ascending ID order (avoids deadlocks):
  `UPDATE product_variants SET reserved = reserved + $q WHERE id = $id AND stock - reserved >= $q`. If any row fails, roll back and show which item is unavailable.
- **On payment:** `stock -= q, reserved -= q`.
- **On release** (session expired, admin cancel, sweeper): `reserved -= q`.
- Stripe sessions expire after 30 minutes (the minimum). The sweeper only releases after 35 minutes.
- Every change writes a `stock_movements` row (reason: `reserve`, `release`, `sale`, `return`, `adjustment`, `restock`).
- Per-variant `low_stock_threshold` (default from settings) shows warnings on the dashboard.

---

## 5. Money and VAT

- All prices are stored as **gross integer cents, VAT included**. Shipping too.
- `vat_rate_bp` (basis points, 2000 = 20%) is set **per product**, defaulting to the setting `default_vat_rate_bp`.
- Every product has a Н-18 tax group (`Б` for 20%).
- **Rounding rule:** VAT is computed per rate at **document level**:
  `net = round(gross_total_for_rate * 10000 / (10000 + rate_bp))`, `vat = gross - net`.
  Per-line net and VAT are allocated for display; any remaining cent goes to the largest line. Totals come from the document-level calculation.
- Orders snapshot: per line `unit_price_cents`, `vat_rate_bp`, `tax_group`; per order `subtotal_cents`, `shipping_cents`, `discount_cents`, `net_cents`, `vat_cents`, `total_cents`, plus the VAT breakdown per rate (jsonb).
- **Currency display:** EUR only, formatted per locale (`12,50 €` in bg). No BGN, no conversion helper, no BGN fields in the schema.
- **Stripe Tax:** not used. Revisit when EU shipping (OSS) arrives.

---

## 6. Fiscal compliance: Наредба Н-18 alternative mode for e-shops [ACC]

Card payments made online are sales that need registration. The shop uses the e-shop alternative mode instead of a fiscal device. This is a dedicated module: `src/modules/fiscal/`.

### 6.1 Before go-live

- Declare the e-shop to НАП (Приложение 33): domain, company, payment methods, payment provider (Stripe), bank account, software, hosting and database location. Report changes within 7 days.
- Store the НАП-assigned e-shop number in settings (`nap_eshop_number`).

### 6.2 Sale document (документ за регистриране на продажба)

Issued automatically when an order becomes `paid`. Contents:

- own **10-digit ascending number**, unique across all operations (`fiscal_documents.number`)
- УНП (unique sale number), format per Н-18 using the НАП e-shop number
- merchant name, EIK, address, contact
- order number and Stripe transaction reference (PaymentIntent ID)
- each line: name (Bulgarian), tax group, quantity, unit price, value; shipping as a line
- total, VAT per tax group, payment method ("карта")
- date and time
- QR code per Приложение 18а (НАП e-shop number, order ID, transaction reference, date, time, total)

Generated as a Bulgarian PDF in the `private` bucket, attached to the order confirmation email, downloadable in admin.

### 6.3 Immutability

- After an order is `paid`, its lines, amounts and fiscal document are never edited or deleted (enforced in code and with a DB trigger that rejects updates to these columns).
- Cancellations, refunds and returns create new records (`fiscal_documents` with type `storno`, linked to the original).

### 6.4 Monthly audit file

- Admin page "Audit file": choose a month, generate the XML (Приложение 38), validate it against the official XSD in a unit test, download it.
- Deadline: by the 15th of the following month according to current sources; confirm **[ACC]**.

### 6.5 Numbering

All counters (fiscal documents, invoices, credit notes) use a `counters` table row locked with `SELECT ... FOR UPDATE` inside the same transaction that creates the document. No Postgres `SEQUENCE` (it leaves gaps on rollback).

---

## 7. Invoices [ACC]

- Invoices are issued **only on request** (the checkbox "I need an invoice"). Consumers who don't ask get the Н-18 document.
- Invoice data collected at checkout: company name, EIK/BULSTAT, VAT number (optional), МОЛ, registered address.
- **Numbering:** 10-digit sequential, using a **dedicated series agreed with the accountant** (`settings.invoice_series_start`). If the accountant prefers, the in-app generator is disabled and invoices are issued in their software; the admin then attaches the PDF to the order.
- Issued at payment (prepayment is the tax event) and in any case within 5 days.
- Language Bulgarian (optionally bilingual bg/en), amounts in EUR, contents per чл. 114 ЗДДС.
- **Credit notes** (кредитно известие) for full or partial refunds, referencing the original invoice number and date.
- PDFs live in the `private` bucket. They are emailed to the customer and downloadable in admin.
- Monthly TSV export of invoices and fiscal documents for the accountant.

---

## 8. Data model (Drizzle, Postgres)

Pattern: base table for language-neutral fields + `*_translations` with primary key `(entity_id, locale)`. Money in integer cents. Timestamps `timestamptz`.

| Table                                           | Fields                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `categories` / `category_translations`          | parent_id, sort, published / name, slug, description, seo_title, seo_description                                                                                                                                                                                                                                                                                                                                          |
| `materials` / `material_translations`           | sort / name, care_instructions                                                                                                                                                                                                                                                                                                                                                                                            |
| `products` / `product_translations`             | sku, status (draft/published/archived), category_id, featured, vat_rate_bp, tax_group, withdrawal_exempt (default false) / name, slug, short_description, description, seo_title, seo_description                                                                                                                                                                                                                         |
| `product_variants`                              | product_id, material_id, size, sku, price_cents, stock, reserved, low_stock_threshold, weight_grams, sort                                                                                                                                                                                                                                                                                                                 |
| `product_images` / `product_image_translations` | product_id, r2_key, width, height, sort / alt                                                                                                                                                                                                                                                                                                                                                                             |
| `stock_movements`                               | variant_id, delta_stock, delta_reserved, reason, order_id, admin_user_id, note, created_at                                                                                                                                                                                                                                                                                                                                |
| `gallery_items` / `gallery_item_translations`   | r2_key, width, height, sort, published / title, alt, description                                                                                                                                                                                                                                                                                                                                                          |
| `pages` / `page_translations`                   | key (about, faq, shipping, returns, terms, privacy), version / title, slug, body_md                                                                                                                                                                                                                                                                                                                                       |
| `settings`                                      | key, value (jsonb), validated by a typed Zod schema per key                                                                                                                                                                                                                                                                                                                                                               |
| `carriers`                                      | code (econt, speedy), name, tracking_url_template, active                                                                                                                                                                                                                                                                                                                                                                 |
| `carrier_offices`                               | carrier_code, office_code, type (office/locker), name, city, postcode, address, active, synced_at                                                                                                                                                                                                                                                                                                                         |
| `orders`                                        | id (uuid), number (human, e.g. `2026-000123`), status, payment_method (`card`; later `cod`), locale, email, phone, full_name, carrier_code, delivery_type, money fields (section 5), vat_breakdown (jsonb), invoice_requested, invoice fields, terms_version, terms_accepted_at, stripe_session_id, stripe_payment_intent_id, tracking_number, confirmation_sent_at, shipped_email_sent_at, admin_notified_at, timestamps |
| `order_addresses`                               | order_id, kind (shipping), snapshot of office (code, name, address) or address fields                                                                                                                                                                                                                                                                                                                                     |
| `order_items`                                   | order_id, line_type (`product`; later `personalized`, `custom_quote`), product_id and variant_id (nullable FK, on delete set null), name snapshot (bg and order locale), variant_label, sku, unit_price_cents, quantity, vat_rate_bp, tax_group, withdrawal_exempt, personalization (jsonb, null in Phase A)                                                                                                              |
| `order_events`                                  | order_id, from_status, to_status, actor (system/admin id/stripe), note, created_at                                                                                                                                                                                                                                                                                                                                        |
| `fiscal_documents`                              | number (10-digit), unp, order_id, type (sale/storno), refers_to_id, r2_key, qr_payload, issued_at                                                                                                                                                                                                                                                                                                                         |
| `invoices`                                      | number (10-digit), series, order_id, type (invoice/credit_note), refers_to_id, r2_key, issued_at                                                                                                                                                                                                                                                                                                                          |
| `counters`                                      | name (fiscal, invoice), value                                                                                                                                                                                                                                                                                                                                                                                             |
| `withdrawals`                                   | order_id, email, items (jsonb, optional), submitted_at, acknowledged_at, status (received/refunded/rejected), note                                                                                                                                                                                                                                                                                                        |
| `stripe_events`                                 | event_id (unique), type, processed_at                                                                                                                                                                                                                                                                                                                                                                                     |
| `email_outbox`                                  | to, template, locale, payload (jsonb), status, attempts, last_error, sent_at                                                                                                                                                                                                                                                                                                                                              |
| `contact_messages`                              | name, email, message, created_at (deleted after 12 months)                                                                                                                                                                                                                                                                                                                                                                |
| `admin_audit_log`                               | admin_user_id, action, entity, entity_id, diff (jsonb), created_at                                                                                                                                                                                                                                                                                                                                                        |
| Better Auth tables                              | user, session, account, verification, two_factor, rate_limit                                                                                                                                                                                                                                                                                                                                                              |

Rules:

- Products and variants referenced by orders are never hard-deleted, only archived.
- Phase C will add a `quotes` table and pay through a Checkout Session per quote (not reusable Payment Links).

### Order status machine

```
pending ──paid──> paid ──> in_production ──> shipped ──> delivered (manual)
   │                │
   ├─expired        ├─> refunded / partially_refunded
   └─cancelled      └─> paid_needs_attention (paid after release, stock not available)
```

Every transition is a guarded update (`... WHERE status = <expected>`) and writes an `order_events` row.

---

## 9. Checkout flow

1. **Cart** (localStorage via Zustand, every access in try/catch) holds only `{variantId, quantity}`. The server validates it with Zod and loads variants from the database. Browser prices are never trusted.
2. **Delivery step** (section 3) and optional invoice details.
3. **Review step**: full order summary (items, shipping, total, VAT), delivery time, link to terms, withdrawal information, checkbox accepting the terms. The submit button reads **"Поръчка с задължение за плащане"** (en: "Order with obligation to pay") **[LAW]**.
4. **Server Action** (rate-limited per IP):
   - one transaction: create `pending` order + `order_items` snapshot + `order_addresses`, reserve stock, compute shipping and VAT
   - create Stripe Checkout Session: `price_data` line items from the database, one shipping option, `locale` from the order locale, `customer_email`, `payment_method_types: ['card']`, `metadata.orderId`, `payment_intent_data.metadata.orderId`, `expires_at` = now + 30 min, idempotency key = `checkout-<orderId>`
   - store `stripe_session_id`, redirect
   - if session creation fails, release the reservation and show an error
5. **Webhook** `POST /api/stripe/webhook`:
   - read the raw body with `await req.text()`, verify with `constructEventAsync`
   - in **one transaction**: `INSERT INTO stripe_events ... ON CONFLICT DO NOTHING`; if already present, return 200; otherwise apply the state change
   - side effects (PDFs, emails) run **after commit**, each guarded by its `*_sent_at` / existence check so retries are safe
   - `checkout.session.completed` (with `payment_status = paid`):
     - order `pending`: mark `paid`, convert reservation to sale, store PaymentIntent ID, issue fiscal document, issue invoice if requested, queue confirmation email (customer locale, with fiscal document, invoice, terms PDF and withdrawal form attached) and admin notification
     - order `expired`/`cancelled`: try to reserve again; if it succeeds continue as above, otherwise set `paid_needs_attention` and notify the admin (one-click refund in admin)
   - `checkout.session.expired`: `pending` to `expired`, release stock
   - `charge.refunded`: find the order by PaymentIntent ID; full or partial refund status, storno fiscal document, credit note if an invoice exists, stock return only if the admin marks items as returned
   - `charge.dispute.created`: flag the order and notify the admin
6. **Success page** fetches the session from Stripe by `session_id` and shows paid / processing / failed, independent of the webhook.
7. **Admin cancel** of a `pending` order calls `stripe.checkout.sessions.expire` first, then releases stock.

---

## 10. Features

### 10.1 Admin (`/admin`, English UI, outside `[locale]`)

- **Access:** login with mandatory TOTP 2FA, backup codes, rate limiting. Guard per section 2.3.
- **Dashboard:** today's and this month's orders and revenue, recent orders, orders needing attention, low-stock warnings, missing translations.
- **CRUD:** categories, materials, products, variants, gallery, pages. One tab per locale with a "translation complete" indicator.
- **Images:** drag-and-drop. The server issues a presigned PUT with a server-generated key, signed `Content-Type` (jpeg, png, webp, avif only; no SVG) and `Content-Length` (max 15 MB). The browser uploads directly to R2. The server records key and dimensions. Reorder supported; alt text required in `bg`.
- **Orders:** list with status filters and search, detail view with event history, status transitions, tracking entry (sends email), document downloads, cancel, refund link to Stripe, "paid needs attention" handling.
- **Withdrawals:** list of submitted withdrawals, mark refunded or rejected.
- **Fiscal:** audit file generation (6.4), monthly TSV export for the accountant.
- **Settings:** shipping rates and threshold, company details (invoices, footer, documents), НАП e-shop number, invoice series, VAT default, low-stock default.
- **Stock:** manual adjustment with reason, movement history.
- **Launch checklist:** products without real photos, missing `bg` translations, missing legal pages, missing НАП number, Stripe in test mode.
- **Cache:** every save calls `updateTag()` (Next 16 read-your-own-writes) for the affected tags.
- Every write is recorded in `admin_audit_log`.

### 10.2 Storefront (`/[locale]/...`, `bg` unprefixed)

| Page             | Content                                                                                              |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| Home             | Hero, featured products, categories, gallery teaser, how it works, trust signals                     |
| Shop             | Product grid, filter by category and material, sort by price or newest                               |
| Category         | Landing page with SEO text                                                                           |
| Product          | Image gallery, variant picker, stock status, description, care, delivery time, add to cart           |
| Gallery          | Finished work                                                                                        |
| Cart             | Slide-over sheet only, with a "Checkout" button                                                      |
| Checkout         | Delivery, invoice details, review (section 9)                                                        |
| Success / cancel | Order state or recovery                                                                              |
| Withdrawal       | "Отказ от договора тук" function (10.3)                                                              |
| Content pages    | About, FAQ, shipping, returns, terms, privacy (Markdown from CMS)                                    |
| Contact          | Form with Turnstile, sent through Resend. Static map image linking to a map; no embedded map iframe. |

- Localized slugs; the language switcher maps the current page to the target locale's slug. Missing translations fall back to `bg`.
- Draft content never appears publicly.

### 10.3 Withdrawal function [LAW]

Required since 19 June 2026 (Directive 2023/2673). Bulgarian transposition was late; confirm the current ЗЗП wording.

- A clearly labelled link **"Отказ от договора тук"** in the footer, in order emails and on the order success page.
- Step 1: order number + email (+ optional item selection).
- Step 2: confirmation screen, button to confirm withdrawal.
- Result: `withdrawals` row, on-screen acknowledgement and an email acknowledgement with date and time of receipt.
- The model withdrawal form (PDF) stays available on the returns page.
- Items with `withdrawal_exempt = true` (Phase B personalized goods) show the exemption reason instead.

### 10.4 Emails (React Email, localized, sent through `email_outbox`)

- Order confirmation (with fiscal document, invoice if any, terms PDF, withdrawal form)
- Shipped, with tracking link
- Withdrawal acknowledgement
- Refund issued (with credit note if any)
- Admin: new order, order needs attention, dispute, withdrawal received, audit file reminder
- Contact form message

Sending domain with SPF, DKIM and DMARC.

### 10.5 Placeholder content

The seed script creates realistic sample categories, materials, products and gallery items in `bg` and `en` with neutral placeholder images. The admin seed is idempotent (creates the admin only if absent) and forces 2FA enrolment at first login.

---

## 11. Security

- Zod validation on every Server Action and route handler input (Server Actions are public endpoints).
- Rate limiting: Better Auth `rateLimit.storage: "database"` and `verification.storeInDatabase: true`; a Cloudflare WAF rate-limit rule on `/api/auth/*`; Workers Rate Limiting binding for checkout, contact and withdrawal actions.
- Turnstile on contact and withdrawal forms, verified server-side.
- CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `frame-ancestors 'none'`.
- Secrets in `wrangler secret`. Stripe **restricted** API key with only the needed permissions.
- R2 CORS on the `media` bucket limited to the shop origin. The `private` bucket is never public; files are served through authenticated admin routes or short-lived signed URLs.
- Optional defence in depth: Cloudflare Access in front of `/admin`.
- Dependency security: Renovate groups Next.js and OpenNext and never auto-merges them, but vulnerability alerts are on and security PRs are handled the same day.

---

## 12. Legal, privacy, accessibility, SEO

### Legal [LAW]

- Terms, privacy policy, returns and shipping pages in `bg` (authoritative) and `en`.
- Company identification (name, EIK, VAT number, address, contact) in the footer and terms.
- Pre-contract information per ЗЗП: product characteristics, total price, delivery cost and time, payment, withdrawal right and its exceptions, 2-year legal guarantee of conformity.
- **No ODR link** (the EU platform closed on 20 July 2025). Instead: information on alternative dispute resolution (КЗП conciliation commissions).
- 14-day withdrawal right for standard items, withdrawal function (10.3) and model form.
- Terms are versioned (`pages.version`); the accepted version is stored per order.

### Privacy (GDPR)

- Processors and recipients listed in the privacy policy: Cloudflare, Neon, Stripe, Resend, Econt, Speedy, and the accountant.
- EU data regions where available. DPAs accepted with every processor.
- Retention: orders, fiscal documents and invoices 10 years **[ACC]**; contact messages 12 months; Better Auth sessions per expiry.
- Data subject requests handled manually by the admin; documented procedure in the README.
- No cookie banner: only cookieless analytics, strictly necessary localStorage (cart) and admin auth cookies. No third-party embeds that set cookies.

### Accessibility

WCAG 2.1 AA (keyboard navigation, focus states, contrast, alt text). The European Accessibility Act likely exempts the business as a microenterprise **[LAW]**; the target stays AA.

### SEO

- `generateMetadata` per page and locale, canonical URLs, hreflang alternates, `sitemap.ts` from the database, `robots.ts`.
- OG image = main product image (no dynamic OG generation).
- JSON-LD: Organization/LocalBusiness, Product with Offer (price in EUR, availability), BreadcrumbList.
- Lighthouse 90+ on mobile.

---

## 13. Quality

### Tests

- **Vitest unit:** pricing, VAT rounding (1-cent edge cases), shipping threshold (4999, 5000, 5001), cart validation, slug and locale helpers, order status transitions, fiscal document and QR payload builder, audit file XML against the official XSD.
- **Vitest integration** (Postgres in Docker): order creation, concurrent reservation race (N parallel checkouts for the last unit), webhook idempotency (same event twice, out of order), paid-after-expiry path, counter numbering under concurrency (no gaps, no duplicates), admin actions reject unauthenticated calls.
- **Playwright E2E:** browse, cart, delivery step, review step, redirect to Stripe (asserted, not completed), success page driven by signed webhook fixtures; admin login with TOTP, product editing; withdrawal flow.
- **Nightly:** one full Stripe test-mode checkout through the hosted page.
- **Release smoke test** in the real Workers runtime (`opennextjs-cloudflare preview`).

### Tooling

ESLint, Prettier, Husky + lint-staged, strict TypeScript (`noUncheckedIndexedAccess`), Renovate (see 11).

### CI/CD

- PRs: lint, typecheck, unit, integration, build, bundle size check.
- `main`: migrations against the **direct** Neon connection (not Hyperdrive), then `wrangler deploy`.
- Branch protection on `main`; all work through PRs.

### Observability

Workers Logs on. Error tracking (Sentry Workers SDK, after checking OpenNext bundling compatibility). Stripe webhook failure emails. External uptime check.

### Backups

Nightly `pg_dump` to the `private` bucket with a lifecycle rule. One restore test before launch, documented in the README.

### Documentation

README with architecture diagram, local setup (`docker compose up`), deploy and restore instructions. Short ADRs for the major decisions (including Hyperdrive split, Н-18 alternative mode, invoice series, bg-first i18n, no dual pricing). `.env.example`.

---

## 14. Milestones

| #   | Milestone   | Done when                                                                                                                                                                                     | Estimate    |
| --- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| A1  | Skeleton    | Next + OpenNext versions pinned and verified (locale proxy works, build passes), page live on the custom domain, query through both Hyperdrive configs, R2 read, D1 tag cache wired, CI green | 2 to 3 days |
| A2  | Data + i18n | Schema migrated, seed in bg + en, locale routing and switcher                                                                                                                                 | 2 to 3 days |
| A3  | Admin       | Auth with mandatory 2FA, CRUD for catalog, images, gallery, pages, settings, audit log                                                                                                        | 6 to 8 days |
| A4  | Storefront  | All public pages localized and cached                                                                                                                                                         | 4 to 5 days |
| A5  | Checkout    | Delivery step with office sync, review step, reservation, Stripe, webhooks, sweeper, emails via outbox, order management                                                                      | 6 to 8 days |
| A6  | Compliance  | Fiscal documents, audit file, invoices and credit notes, withdrawal function, legal pages, accountant TSV export                                                                              | 5 to 8 days |
| A7  | Launch      | НАП declaration filed, SEO, Lighthouse and accessibility pass, backup restore tested, accountant and lawyer sign-off, Stripe live mode                                                        | 2 to 3 days |

Total: roughly 6 to 8 weeks full-time.

---

## 15. Open confirmations

### Accountant [ACC]

1. Н-18 alternative mode applies to this shop with Stripe as the payment provider; required document contents and УНП format (6.2).
2. Audit file format version and deadline (6.4).
3. Invoice series and whether invoices are issued in-app or in the accountant's software (7).
4. Retention periods for orders, fiscal documents and invoices (12).
5. For Phase A+: whether card (alternative mode) and cash on delivery via ППП can be combined, and what document ППП sales need.

### Lawyer [LAW]

1. Current ЗЗП wording for the withdrawal function (10.3).
2. Order button text and pre-contract information (9, 12).
3. ADR information wording (12).
4. Terms, privacy policy and returns texts.
5. Whether the European Accessibility Act microenterprise exemption applies.

---

## 16. Out of scope for Phase A

- Cash on delivery (Phase A+, model already supports `payment_method = cod`).
- Personalization (Phase B) and custom quotes (Phase C).
- Shipping outside Bulgaria, carrier API labels, tracking sync.
- Discount codes (when added: show the lowest price of the last 30 days next to any reduction), customer accounts, reviews.
- Locales beyond bg and en.
- Dual BGN/EUR price display.
- AI translation drafts, dynamic OG images, embedded maps.
