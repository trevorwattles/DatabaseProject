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

1. **Ledger, not a counter.** `stock_movements` is append-only. On-hand = `SUM(qty_delta)`. `inventory_items.on_hand` is only a cache, updated in the same transaction. Fix mistakes with new movements, never UPDATE/DELETE old ones.
2. **Base units + NUMERIC.** Store every quantity in the ingredient's base unit as `NUMERIC`. Never use float/real/double for quantities or money. Convert purchase/recipe units through `units` / `unit_conversions`.
3. **Idempotency.** `webhook_events.event_id` is UNIQUE (insert with ON CONFLICT DO NOTHING). Sale/refund movements have a partial unique index on `(source_type, source_id, ingredient_id)`. Sum per ingredient before inserting.
4. **Raw vs. derived.** Store Square data as received (`webhook_events`, `pos_orders`, `pos_order_lines`, `pos_order_line_modifiers`). Movements are derived and must be reproducible from raw data.
5. **Surface unmapped items.** Sales with no recipe go into `unmapped_pos_items`. Never silently drop them.
6. **Weighted average cost.** Store `unit_cost` on delivery movements and update `inventory_items.avg_cost` on receipt.
7. **Tenancy.** Every business row carries `organization_id` (and `location_id` where relevant). Every query must be scoped through the data-access layer. RLS is a stretch goal.
8. **Webhook security.** Verify `x-square-hmacsha256-signature` before storing. Return 200 fast and process async. Square order webhooks lack line items, so fetch the order via the Orders API.

## MVP scope

Auth + one org/location → Square OAuth + menu import → ingredients/units/suppliers → recipe mapping (with modifiers) → webhook → idempotent sale movements → dashboard (on-hand, low stock, unmapped) → mobile waste logging → physical count + variance report → auto-drafted POs + receiving → seed script + sales simulator.

Stretch only if ahead: AI invoice scanning, emailing POs, NL questions, forecasting, multi-location, prep batches, RLS.

## Repo conventions

- **Planning docs live in `docs/`**. `docs/PROPOSAL.md` is the source of truth for the ERD, flows, and schedule. Wireframes are in `docs/ux/`.
- `PROJECT_LOG.md`: add a row (date, what, hours) for every work session and update the total.
- Commit and push after each meaningful step. Conventional-style messages (`feat:`, `fix:`, `docs:`, `chore:`).
- Secrets go in `.env.local` (gitignored). Commit only `.env.example` with placeholder values.
- Table and column names: `snake_case`, plural tables, `uuid` PKs, `timestamptz` timestamps.
- Mermaid diagrams must render on GitHub: simple identifiers, quoted labels.
