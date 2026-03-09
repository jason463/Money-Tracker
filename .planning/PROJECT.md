# Money Tracker

## What This Is

A dead-simple shared weekly spending tracker for two people (Jason and Shelby). Manual entry of discretionary purchases with a shared weekly budget that auto-resets every Monday. Mobile-first PWA accessible via shared URL — no login required.

## Core Value

Jason and Shelby can see how much of their weekly discretionary budget remains at a glance, with zero friction to log a purchase.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Manual expense entry: amount, label, who spent it (Jason or Shelby)
- [ ] Shared weekly discretionary budget, editable in the UI by either person
- [ ] Running total spent and remaining budget displayed prominently
- [ ] Visual budget indicator: color-coded progress bar + colored remaining text (green → yellow → red)
- [ ] Auto-reset every Monday (new week starts fresh)
- [ ] Rolling history of recent weeks (keep ~4 weeks, auto-delete older)
- [ ] Either person can delete any entry (full trust model)
- [ ] Shared access via URL, no authentication or login
- [ ] Mobile-first responsive design, installable as PWA (home screen icon)

### Out of Scope

- Bank connections or transaction imports — manual-only by design
- Categories, tags, or analytics — this is a simple tracker, not a budgeting tool
- Recurring bills, subscriptions, rent, or utilities — discretionary spending only
- Budgeting by category — one shared pot, no sub-budgets
- User accounts or authentication — shared URL, full trust
- Multiple budgets or households — two people, one budget

## Context

- Built for personal use by Jason and Shelby to track shared discretionary spending
- The "who" selector is hardcoded to Jason and Shelby (not configurable users)
- Entries are quick and minimal: tap, type amount, quick label, pick name, done
- Week boundary is Monday 12:00 AM (local time or a fixed timezone)
- Old weeks kept for casual reference, not for reporting or analytics
- No sensitive data — amounts and labels only, no auth needed

## Constraints

- **Tech stack**: Next.js (App Router), Tailwind CSS, Postgres database (Supabase or Vercel Postgres — whichever is simpler)
- **Deployment**: Vercel via CLI, GitHub repo for source
- **Design**: Mobile-first, must work well on phone screens as primary device
- **PWA**: Must be installable to home screen with app-like experience
- **Simplicity**: Minimal UI, minimal features, resist scope creep

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| No auth/login | Full trust between two people, zero friction | — Pending |
| Hardcoded Jason/Shelby | Not a multi-tenant app, just two people | — Pending |
| Weekly reset on Monday | Aligns with typical pay/budget cycles | — Pending |
| Rolling 4-week history | Casual reference without unbounded data growth | — Pending |
| Supabase or Vercel Postgres | Pick whichever is simpler to set up | — Pending |
| Budget editable in UI | Flexibility without needing code changes | — Pending |

---
*Last updated: 2026-03-09 after initialization*
