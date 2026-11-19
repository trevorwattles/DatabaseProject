# Claude Code Prompt — CS 452 Final Project: Initial Planning Phase

Copy everything below the line into Claude Code (run it from the root of your empty repo).

---

You are helping me plan my BYU CS 452 (Database Modeling Concepts) final project. The repo is empty. Read this entire prompt before you start, then work through the deliverables in order.

**IMPORTANT: scope of this session.** This session is ONLY the initial planning submission described in Section 1: a summary with pictures, an initial ERD, a rough system design diagram, daily goals, and optional UX sketches. The assignment says this should take 10–90 minutes and capture *initial thoughts* that will be refined later, so aim for clear and rough, not exhaustive. **Do NOT** scaffold the app, install packages, set up a database, create migrations, or write any application code. Sections 2–4 describe the full project so the planning docs are accurate. They are context, not tasks for now.

## 1. The assignment (what I must submit)

I submit a link to **one markdown file in this public GitHub repo** that contains:

1. **Summary of the project's purpose and goals**, in words and pictures, so classmates can quickly grasp the project
2. **Initial ERD** sketching the data the project will use
3. **Rough system design diagram**: technologies as shapes, interactions as arrows
4. **Initial daily goals** from now (Thursday, Nov 19, 2026) through the end of class (assume the last day of class is **Thursday, Dec 10, 2026**; leave a note at the top of that section so I can confirm the date)
5. **Optional: rough UX sketches**. I want these included.

Overall project requirements from the course:
- Build something I'm genuinely interested in
- It must **read and write data** (SQL, NoSQL, API, files, etc.)
- Use AI to help build something better than I could alone in the same time
- **30–40 hours** of total work

## 2. The project idea

**Working name: ParLevel** (placeholder, I may rename it). It's an inventory tracking and auto-purchasing platform for small restaurants and small franchise locations, driven by their POS sales data.

### The problem
Independent restaurants, cafés, and small franchise owners (1–5 locations) mostly track inventory with paper count sheets, spreadsheets, or a manager eyeballing the walk-in. That leads to:
- Running out of key ingredients mid-shift (lost sales)
- Over-ordering and spoilage (wasted money)
- No visibility into **where money leaks**: waste, over-portioning, theft, comped meals
- Hours a week spent on manual counts and building supplier orders

Existing tools (MarketMan, MarginEdge, Restaurant365, xtraCHEF) are often too expensive, too complex to set up, or built for larger operations. The opening is a **simple, fast-to-set-up, affordable** tool for small operators.

### How it works (core loop)
1. **Connect POS.** The owner connects their Square account (OAuth). The app imports their menu (Catalog API) so they don't type it in by hand.
2. **Map recipes.** Each menu item variation maps to ingredients and quantities. Example: "Cheeseburger" = 1 bun, 4 oz beef patty, 1 slice cheddar, 0.5 oz sauce. Modifiers adjust usage ("extra cheese" +1 slice, "no onion" −0.25 oz onion). Prep items (sauces, dough, chopped veg) are their own recipes that turn raw ingredients into a prepped ingredient with a batch yield.
3. **Sales deplete inventory automatically.** Square webhooks (order/payment events) arrive in real time. Each completed order line is expanded through the recipe and its modifiers into ingredient usage, which is written as **sale** movements in the stock ledger. Refunds and cancellations write reversing movements.
4. **Deliveries add stock.** Receiving a purchase order or entering a supplier invoice writes **delivery** movements. (Stretch: snap a photo of the invoice and AI extracts the line items for the user to confirm.)
5. **Waste logging.** Staff quickly log waste from a phone (dropped, expired, burnt, comped), which writes **waste** movements with a reason.
6. **Physical counts correct drift.** Weekly counts (or daily for high-value items like proteins) compare *expected* on-hand against *actual* counted. The difference is written as a **count_adjustment** movement and shown in a **variance report** in units and dollars.
7. **Reorder automatically.** When on-hand drops below an item's reorder point, the system drafts a purchase order grouped by supplier, using par levels, pack sizes, and recent usage. A manager reviews and sends it (stretch: email the PO to the supplier).

### Why owners would use it
The headline feature is the **variance report**: "You lost $1,240 in chicken last month that sales don't explain." Automatic reordering and time saved are secondary benefits. It shows owners exactly where money is leaking.

### Positioning
- **Target customer (beachhead):** independent quick-service restaurants and cafés, and small franchise owners, with 1–5 locations, **using Square POS**, starting in Utah County
- **Differentiation:** setup in under an hour (menu import plus recipe templates for common items), mobile-first counting, plain-English dollar insights
- **Go-to-market:** in-person visits to local shops, a pilot with 1–3 friendly owners, then franchise owner networks and the Square App Marketplace
- **Expansion path:** Clover next, then Toast via their partner program; multi-location transfers; supplier integrations; AI demand forecasting
- **Key risks to note:** POS API access and approval (Toast is gated; Square and Clover are open), messy recipe data, owner trust with sales data, and support load

## 3. Recommended technical direction

Use this as the default architecture in the docs. Briefly note alternatives where reasonable.

- **Frontend:** Next.js (App Router) + TypeScript + Tailwind CSS + shadcn/ui; responsive / PWA so counts and waste logging work well on phones
- **Backend:** Next.js route handlers / server actions (one deployable app). The webhook endpoint verifies the Square signature (`x-square-hmacsha256-signature`).
- **Database:** **PostgreSQL** (Neon or Supabase) with **Drizzle ORM** or Prisma and versioned migrations. The data is highly relational, and the class focus is data modeling.
- **POS integration:** Square **sandbox** (free) for development: OAuth, Catalog API (menu import), Orders API, and webhooks (e.g. `order.created`, `order.updated`, `payment.updated`, `refund.created`, `catalog.version.updated`)
- **Background work:** a simple job/queue approach (e.g. a `jobs` table or Vercel Cron) for processing webhook events, nightly reorder checks, and token refresh
- **Auth & multi-tenancy:** Auth.js / Clerk / Supabase Auth; every business row is scoped by `organization_id` (and `location_id` where relevant); consider Postgres Row-Level Security
- **AI features (product):** Claude API for invoice photo → line items (stretch), natural-language questions ("what did I waste most last week?"), and later simple usage forecasting
- **Email:** Resend (stretch: send POs to suppliers)
- **Hosting:** Vercel + Neon/Supabase

### Key data-design principles (important for this class; explain these in the docs)
- **Ledger, not a counter.** Inventory is an **append-only `stock_movements` ledger** (sale, delivery, waste, count_adjustment, prep_in, prep_out, refund_reversal, transfer). Current on-hand = sum of movements, optionally cached in `inventory_items.on_hand` (via trigger or app logic) for fast reads. This gives a full audit trail and makes variance reports possible.
- **Base units + conversions.** Store every quantity in the ingredient's base unit as `NUMERIC` (never float). Handle purchase units (case of 6 × #10 cans, 50 lb bag) and recipe units (oz, each, cup) through a conversion table, including ingredient-specific conversions.
- **Idempotency.** Webhooks can arrive twice or out of order. Store raw events with a unique Square `event_id`, then process into movements with a uniqueness rule on (source_type, source_id, ingredient_id).
- **Keep raw POS data separate from derived data.** Imported orders and order lines are stored as received. Movements are derived from them, so they can be reprocessed if a recipe mapping changes.
- **Surface unmapped items.** If a sale arrives for a menu item or modifier with no recipe, record it and flag it in the UI instead of silently ignoring it.
- **Money and history.** Store unit cost on delivery movements so dollar variance reflects real cost (weighted average or last cost; pick one and justify it).

### Suggested entities for the ERD (refine as you see fit)
- `organizations`, `locations`, `users`, `memberships` (role: owner / manager / staff)
- `pos_connections` (location, provider, merchant_id, encrypted tokens, expiry)
- `units`, `unit_conversions`
- `ingredients` (org catalog: name, category, base_unit, is_prepped)
- `inventory_items` (location + ingredient: par_level, reorder_point, on_hand cache, avg_cost)
- `suppliers`, `supplier_items` (supplier, ingredient, SKU, pack size/unit, price, lead time)
- `menu_items`, `menu_item_variations` (with POS ids)
- `recipes`, `recipe_lines` (a recipe belongs to a menu variation **or** produces a prepped ingredient with a yield)
- `modifiers`, `modifier_recipe_lines` (positive or negative quantity deltas)
- `webhook_events` (raw payload, event_id unique, status, processed_at, error)
- `pos_orders`, `pos_order_lines`, `pos_order_line_modifiers`
- `stock_movements` (location, ingredient, qty_delta, movement_type, source_type, source_id, unit_cost, occurred_at, created_by)
- `waste_logs` (reason, notes, photo)
- `inventory_counts`, `inventory_count_lines` (expected, counted, variance qty, variance $)
- `purchase_orders`, `purchase_order_lines` (status: draft / sent / partially_received / received)
- `invoices`, `invoice_lines` (stretch: image, AI-extracted fields, confidence, confirmed flag)
- `unmapped_pos_items`

## 4. Scope for the class (30–40 hours)

**MVP (must finish by the end of class):**
1. Auth + one organization/location
2. Square sandbox connection + menu import
3. Ingredient, unit, and supplier management
4. Recipe mapping UI (including modifiers)
5. Webhook ingestion → idempotent processing → sale movements in the ledger
6. Inventory dashboard (on-hand, low stock, unmapped items)
7. Waste logging (mobile-friendly)
8. Physical count flow + variance report (units and $)
9. Low-stock detection → auto-drafted purchase order grouped by supplier; receiving a PO writes delivery movements
10. Seed script + a sales simulator (creates sandbox orders, or generates fake orders locally) so the demo works without a real restaurant

**Stretch goals (only if ahead of schedule):** AI invoice scanning, emailing POs, natural-language inventory questions, usage forecasting, multi-location, prep batch production.

Mark clearly in the docs what is MVP and what is stretch.

## 5. Deliverables to create in this session (planning docs only)

Create these files:

1. **`docs/PROPOSAL.md`**: the file I'll submit. Sections, in this order:
   - Title, one-line pitch, and a short "TL;DR for classmates" (3–4 sentences)
   - **Problem & solution**, including a **"How it works" diagram** (Mermaid flowchart: Sale on POS → Webhook → Recipe expansion → Ledger → Dashboard / Alerts → Purchase order → Delivery → Ledger)
   - **Goals**: learning goals (data modeling, ledger design, webhooks, idempotency, multi-tenant SQL, AI-assisted dev) and product goals
   - **Target user & business case**: customer, value prop, competition table, go-to-market, risks. Keep it tight, and don't include pricing or billing plans.
   - **Initial ERD** as a Mermaid `erDiagram` with PKs, FKs, key columns, and cardinalities, followed by short notes on the key design decisions (ledger, units, idempotency, raw vs derived, tenancy)
   - **System design diagram** as a Mermaid `flowchart` with the browser/PWA, Next.js app, webhook endpoint, job worker, Postgres, Square (OAuth, Catalog, Orders, Webhooks), Claude API (stretch), and Resend (stretch). Label the arrows with what flows across them.
   - **Key flows** as 2–3 Mermaid `sequenceDiagram`s: (a) order webhook → inventory depletion, (b) physical count → variance, (c) low stock → draft PO → receive
   - **UX sketches**: low-fidelity wireframes for Dashboard, Recipe Mapping, Count (mobile), Waste Log (mobile), Variance Report, Purchase Order. Do these as simple SVG wireframes saved in `docs/ux/` and embedded in the markdown with relative image links (or ASCII-box wireframes in code blocks if SVG gets too slow). Keep them grayscale and rough. They're sketches, not final designs.
   - **MVP vs stretch scope**
   - **Daily goals**: a table with columns Date | Goal | Est. hours | Done ☐, from Thu Nov 19 through Thu Dec 10, 2026. Totals must add up to 30–40 hours. Make Sundays rest days and Thanksgiving (Thu Nov 26) a rest day; keep Wed Nov 25 and Fri Nov 27 light. Front-load the risky parts (Square OAuth/webhooks, schema + migrations) in the first week. Leave buffer days near the end for polish, demo data, README, and the final presentation. Include a "Confirm end-of-class date" note at the top of this section.
   - **Open questions / decisions** I still need to make
2. **`README.md`**: short project overview with the pitch, a link to `docs/PROPOSAL.md`, the planned tech stack, and a "Status: planning phase" badge/line
3. **`CLAUDE.md`**: concise project context for future Claude Code sessions: what the product is, the stack, data-design principles (ledger, base units, NUMERIC, idempotency, tenancy scoping), MVP scope, repo conventions, and "planning docs live in `docs/`"
4. **`docs/ux/*.svg`**: the wireframe files (if using SVG)
5. **`.gitignore`** for a Node/Next.js project (including `.env*`)

## 6. How to work

- **Don't scaffold the Next.js app or write application code yet.** This session is planning docs only.
- Make sure all Mermaid diagrams use syntax that **renders on GitHub** (keep entity and field names simple, avoid special characters in labels, and use quoted labels where needed). Re-check each diagram block for syntax errors before finishing.
- Write for classmates: scannable headings, short paragraphs, diagrams before walls of text. Keep `PROPOSAL.md` readable in about 5 minutes, apart from the ERD and the schedule.
- Where you make a judgment call (e.g. cost method, ORM choice), state the choice and a one-line reason so I can change it.
- When finished:
  1. Show me a summary of the files created and any open questions
  2. Make sure there are no secrets in the repo
  3. Commit with a clear message (e.g. `docs: initial project proposal, ERD, system design, UX sketches, schedule`) and push to the default branch
  4. Remind me that the repo must be **public** and give me the GitHub URL path to `docs/PROPOSAL.md` so I can submit it
