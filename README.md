# ParLevel

![Status: planning phase](https://img.shields.io/badge/status-planning%20phase-lightgrey)

**POS-driven inventory tracking and auto-reordering for small restaurants.** Connect Square, map your recipes once, and ParLevel depletes inventory with every sale, shows exactly where food cost is leaking, and drafts your supplier orders.

*BYU CS 452 (Database Modeling Concepts) final project. "ParLevel" is a working name.*

## 📄 Planning docs

- **[Project proposal](docs/PROPOSAL.md)**: purpose, ERD, system design, key flows, UX sketches, MVP scope, and daily schedule
- [Project log](PROJECT_LOG.md): hours worked

## Planned tech stack

| Layer | Choice |
|---|---|
| Frontend | Next.js (App Router), TypeScript, Tailwind CSS, shadcn/ui, PWA |
| Backend | Next.js route handlers + server actions |
| Database | PostgreSQL (Neon) + Drizzle ORM, versioned migrations |
| POS integration | Square sandbox: OAuth, Catalog, Orders, webhooks |
| Background jobs | `webhook_events` queue + Vercel Cron |
| Auth | Auth.js |
| AI (stretch) | Claude API (invoice scanning, natural-language questions) |
| Email (stretch) | Resend |
| Hosting | Vercel + Neon |

## Status

**Planning phase.** No application code yet. See the [daily goals](docs/PROPOSAL.md#9-daily-goals) for the build schedule.
