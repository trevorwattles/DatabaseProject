# CLAUDE.md

Context for Claude Code sessions in this repo.

## Product

**ParLevel**: inventory tracking and auto-purchasing for small restaurants (1–5 locations) using **Square POS**. Square sales webhooks deplete inventory through recipe mappings. Waste logs, deliveries, and physical counts also write to the ledger. The headline feature is the **variance report** (expected vs. counted, in units and $). Low stock auto-drafts purchase orders grouped by supplier.

This is a BYU CS 452 (Database Modeling) final project: a 30–40 hour budget, last day of class Thu Dec 10, 2026 (confirmed). Data modeling quality matters most.

## Stack

- Next.js (App Router) + TypeScript + Tailwind + shadcn/ui, responsive PWA
- Next.js route handlers / server actions (single deployable app)
- PostgreSQL on Neon + **Drizzle ORM** with versioned SQL migrations
- Auth.js; Square sandbox (OAuth, Catalog, Orders, webhooks)
- Background work: `webhook_events.status` as a queue + Next.js `after()` + Vercel Cron
- Stretch: Claude API (invoice scanning, NL questions), Resend (email POs)
- Hosting: Vercel + Neon

## Data-design principles (do not violate)

1. **Ledger, not a counter.** `stock_movements` is append-only (a trigger rejects UPDATE/DELETE). On-hand = `SUM(qty_delta)`. `inventory_items.on_hand` is only a cache, updated in the same transaction with an atomic `on_hand = on_hand + delta`, only for rows actually inserted (`RETURNING`). Fix mistakes with a `void` movement plus a new correct entry. `occurred_at` = when it happened, `created_at` = when recorded.
2. **Base units + NUMERIC.** Store every quantity in the ingredient's base unit as `NUMERIC`. Never use float/real/double for quantities or money. Convert purchase/recipe units through `units` / `unit_conversions`.
3. **Idempotency.** `webhook_events.event_id` is UNIQUE (insert with ON CONFLICT DO NOTHING). Movement sources are an exclusive arc (one FK column per source + `CHECK num_nonnulls(...) = 1`), and each source has its own unique index, e.g. `(pos_order_line_id, ingredient_id)` for sales and `(receipt_line_id)` for deliveries. Sum per ingredient before inserting. Receipts carry an `idempotency_key`.
4. **Raw vs. derived.** Store Square data as received (`webhook_events`, `pos_orders`, `pos_order_lines`, `pos_order_line_modifiers`, `pos_refunds`). Movements are derived and must be reproducible from raw data, so recipes are versioned (`effective_from` / `effective_to`) and expansion uses the version in effect at the order's `closed_at`.
5. **Surface unmapped items.** Sales with no recipe go into `unmapped_pos_items`. Never silently drop them.
6. **Weighted average cost.** Store `unit_cost` on every movement (receipt cost for deliveries, current `avg_cost` otherwise) and update `inventory_items.avg_cost` on receipt. If on-hand ≤ 0 before a delivery, reset `avg_cost` to the delivery's unit cost.
7. **Tenancy.** Every business row carries `organization_id` (and `location_id` where relevant). FKs between tenant tables are composite `(organization_id, id)` so cross-org references are impossible. Every query must be scoped through the data-access layer. RLS is a stretch goal.
8. **Webhook security.** Verify `x-square-hmacsha256-signature` before storing. Return 200 fast and process async. Square order webhooks lack line items, so fetch the order via the Orders API.

## MVP scope

Auth (Google) + one org/location + roles → Square OAuth + menu import → ingredients/units/suppliers → versioned recipe mapping (modifiers, sub-recipe expansion) → webhook → idempotent sale movements (refunds stored raw, no restock) → dashboard (on-hand, low stock, unmapped + backfill) → mobile waste logging (with voids) → opening count + physical count + variance report → on-order-aware auto-drafted POs + receipts → seed script + sales simulator (same processing path) → ledger tests.

Stretch only if ahead: AI invoice scanning, emailing POs, NL questions, forecasting, multi-location, prep batches, RLS.

## Decisions (planning phase)

- Completed Square orders edited after completion: MVP does not change inventory; store raw and flag it. Diffing versions is stretch.
- Count variance uses expected on-hand as of submit time (`counted_at`), after draining the location's pending webhook events. Counts are blind (expected hidden while counting).
- The first count at a location is an opening count: it writes `opening_balance` movements and is excluded from variance.
- Refunds don't restock (food was already made). There is no "comped" waste reason (comps are already sales).
- Prepped ingredients expand to raw ingredients at sale time in the MVP. Prep batches are stretch.
- Square connection is per organization (merchant); locations map via `square_location_id`.
- Demo data: local fake-order generator for history + a few real Square sandbox orders for the webhook path.
- Full table in `docs/PROPOSAL.md` section 10.

## Repo conventions

- **Planning docs live in `docs/`**. `docs/PROPOSAL.md` is the source of truth for the ERD, flows, and schedule. Wireframes are in `docs/ux/`.
- `PROJECT_LOG.md`: add a row (date, what, hours) for every work session and update the total.
- Commit and push after each meaningful step. Conventional-style messages (`feat:`, `fix:`, `docs:`, `chore:`).
- Secrets go in `.env.local` (gitignored). Commit only `.env.example` with placeholder values.
- Table and column names: `snake_case`, plural tables, `uuid` PKs, `timestamptz` timestamps.
- Mermaid diagrams must render on GitHub: simple identifiers, quoted labels.
