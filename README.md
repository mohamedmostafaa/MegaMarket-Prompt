# MegaMarket-Prompt
**MegaMarket** — customers buy from multiple stores in one cart; orders split per vendor; admins manage commissions/fees; vendors manage catalogs, inventory, and fulfillment.

You are an elite, ruthless, production-grade full-stack builder.
Deliver a complete **Multi-Vendor Marketplace** (like Amazon/Noon) with Web storefront + Admin + Vendor Portal, Arabic/English (RTL), EGP/USD, local & global payments, shipping per vendor, commission & payouts.

# PRODUCT
**MegaMarket** — customers buy from multiple stores in one cart; orders split per vendor; admins manage commissions/fees; vendors manage catalogs, inventory, and fulfillment.

# STACK (STRICT)
- Web: Next.js 14 (App Router, TypeScript), Tailwind + shadcn/ui, Next SEO, PWA.
- Backend: Next.js Route Handlers (monolith) or NestJS (if you prefer) — pick one and keep it consistent.
- DB: PostgreSQL + Prisma (strict schema + migrations).
- Mobile (optional scaffold): Expo React Native consuming same API.
- Auth: NextAuth (customer/vendor/admin), RBAC.
- Payments:
  - **Stripe Connect** (global) **and** Paymob “Marketplace/Disbursements” (EG) via feature flag.
- Search: Meilisearch (or Algolia toggle).
- Media: Cloudinary.
- Notifications: Email (Resend), Web push, optional Expo push.
- i18n: next-intl (ar/en), dynamic RTL.

# CORE FLOWS
- **Customer**: browse by category/vendor, global search, add to cart (multi-vendor), checkout → cart auto-splits to **Sub-Orders** per vendor; shipping/tax per vendor; track orders & invoices.
- **Vendor Portal**: onboarding (KYC), store profile (logo, banner, policies), products/variants, inventory, pricing & discounts, orders queue, shipping labels, refunds, analytics.
- **Admin**: approve vendors, set commissions (flat/percent/tiers), manage disputes, payouts schedule, featured stores/products, platform CMS blocks, global reports.
- **Payments**: collect from buyer → hold platform fee → route vendor share (Connect/Paymob) → webhooks to mark sub-orders `paid`.
- **Logistics**: shipping method per vendor (own courier or platform), rates & SLA, tracking numbers.
- **Reviews**: store & product ratings (with anti-spam guard).
- **Compliance**: invoices per sub-order; audit log; export data.

# PRISMA SCHEMA (essentials)
model User { id Int @id @default(autoincrement()) email String @unique name String? role Role @default(CUSTOMER) passwordHash String createdAt DateTime @default(now()) Vendor? Customer? }
enum Role { CUSTOMER VENDOR ADMIN }

model Vendor {
  id Int @id @default(autoincrement())
  ownerId Int @unique
  owner   User @relation(fields: [ownerId], references: [id])
  name String
  slug String @unique
  logo String? banner String?
  status VendorStatus @default(PENDING) // PENDING/ACTIVE/SUSPENDED
  kyc JSON?
  products Product[]
  payouts Payout[]
  createdAt DateTime @default(now())
}
enum VendorStatus { PENDING ACTIVE SUSPENDED }

model Product {
  id Int @id @default(autoincrement())
  vendorId Int
  vendor   Vendor @relation(fields:[vendorId], references:[id])
  slug String @unique
  title String title_ar String?
  description String? description_ar String?
  images String[]
  basePrice Decimal @db.Decimal(12,2)
  currency String // EGP/USD
  attributes JSON?
  isActive Boolean @default(true)
  variants Variant[]
  inventory Inventory[]
  createdAt DateTime @default(now())
}

model Variant { id Int @id @default(autoincrement()) productId Int sku String @unique barcode String? attrs JSON? price Decimal? stock Int @default(0) isActive Boolean @default(true) }

model Inventory { id Int @id @default(autoincrement()) productId Int? variantId Int? delta Int reason String? refId String? createdAt DateTime @default(now()) }

model Cart { id String @id userId Int? sessionId String? items CartItem[] currency String subtotal Decimal @db.Decimal(12,2) discounts Decimal @db.Decimal(12,2) total Decimal @db.Decimal(12,2) }
model CartItem { id Int @id @default(autoincrement()) cartId String vendorId Int productId Int variantId Int? qty Int price Decimal @db.Decimal(12,2) }

model Order {
  id String @id
  userId Int
  currency String
  total Decimal @db.Decimal(12,2)
  status OrderStatus @default(PENDING) // PENDING/PAID/CANCELLED/REFUNDED
  subOrders SubOrder[]
  address JSON
  paymentProvider String
  transactionId String?
  createdAt DateTime @default(now())
}
enum OrderStatus { PENDING PAID CANCELLED REFUNDED }

model SubOrder {
  id String @id
  orderId String
  vendorId Int
  items JSON // freeze snapshot
  subTotal Decimal @db.Decimal(12,2)
  shipping Decimal @db.Decimal(12,2)
  platformFee Decimal @db.Decimal(12,2)
  vendorEarnings Decimal @db.Decimal(12,2)
  status SubOrderStatus @default(CONFIRMED) // CONFIRMED/PACKED/SHIPPED/DELIVERED/RETURNED
  trackingNumber String?
  createdAt DateTime @default(now())
}
enum SubOrderStatus { CONFIRMED PACKED SHIPPED DELIVERED RETURNED }

model CommissionRule { id Int @id @default(autoincrement()) vendorId Int? categoryId Int? percent Decimal @db.Decimal(5,2) flat Decimal? active Boolean @default(true) }
model Payout { id Int @id @default(autoincrement()) vendorId Int amount Decimal @db.Decimal(12,2) currency String status PayoutStatus @default(PENDING) externalRef String? createdAt DateTime @default(now()) }
enum PayoutStatus { PENDING IN_PROGRESS PAID FAILED }

model Review { id Int @id @default(autoincrement()) productId Int? vendorId Int? userId Int rating Int @default(5) comment String? createdAt DateTime @default(now()) }

# API SURFACE (REST example)
- /api/auth/*
- /api/vendors (POST onboard, GET list)
- /api/vendors/[slug] (storefront page)
- /api/products, /api/products/[slug]
- /api/cart (GET/POST/PATCH/DELETE) — supports multi-vendor items
- /api/checkout/session (POST) → returns provider session
- /api/webhooks/stripe-connect, /api/webhooks/paymob
- /api/orders (GET user), /api/orders/[id]
- /api/admin/* (vendors, commissions, payouts, disputes)

# CHECKOUT & MONEY FLOW
- Create parent `Order` + derived `SubOrders` per vendor.
- Compute platform fee via `CommissionRule` (percent/flat/tiers).
- Charge buyer once; allocate vendor earnings internally.
- On webhook success → mark Order PAID; SubOrders CONFIRMED; create `Payout` records.
- Payouts cron batches to vendors (manual trigger + schedule).

# UI
- Storefront: Home (featured vendors/products), categories, global search, vendor pages, product PDP, cart, checkout, orders.
- Vendor Portal: dashboard (revenue, orders), product CRUD, inventory, shipping profiles, disputes center, payout history.
- Admin: vendor approvals, commissions, featured placements, payouts, moderation, analytics.

# SEARCH & SEO
- Meilisearch index for products + vendors (ar/en fields).
- SEO: product rich snippets, vendor org schema, breadcrumbs; ISR for product/vendor pages.

# I18N / RTL
- ar/en with next-intl, RTL aware components, currency/numerals localized.

# SECURITY & QUALITY
- RBAC hard guards; validate all inputs with Zod.
- Recompute totals server-side; prevent price tampering.
- Webhook signature verification.
- Logging/audit for financial events.
- Seed: 5 vendors (clothing, supermarket, electronics, etc.), 40 products, demo orders.
- README with env.example (Stripe Connect/Paymob keys), run scripts, and payout flow diagram.
- Test plan: multi-vendor cart → checkout → split → webhook → vendor portal sees orders → mark shipped → payout.

Deliver now: full codebase (monorepo ok), migrations, seed, README, and deployment steps (Vercel + Neon/Railway). No placeholders for money flow or splitting orders.
