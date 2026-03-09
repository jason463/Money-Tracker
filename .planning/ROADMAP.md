# Roadmap: Money Tracker

**Created:** 2026-03-09
**Milestone:** v1.0
**Phases:** 5
**Granularity:** Coarse

## Phase 1: Project Scaffolding & Database

**Goal:** Working Next.js app with Neon Postgres database, schema created, development workflow established.

**Requirements:** INFR-01, INFR-02, INFR-03

**Plans:** 1 plan

Plans:
- [ ] 01-01-PLAN.md -- Scaffold Next.js 15, create database layer, verify connectivity

**Delivers:**
- Next.js 15 App Router project with Tailwind CSS v4
- Neon Postgres database with expenses and settings tables
- Database utility module (`@neondatabase/serverless`)
- Local dev environment with `vercel env pull`

**Success Criteria:**
- [ ] `npm run dev` starts without errors
- [ ] Can insert and query expenses from a server component
- [ ] Settings table has default budget row
- [ ] Environment variables configured for local and Vercel

**Depends on:** Nothing (first phase)

---

## Phase 2: Core UI & Budget Display

**Goal:** Mobile-first three-zone layout with live budget display (hero number + HSL progress bar).

**Requirements:** BUDG-01, BUDG-02, BUDG-03, MOBI-01, MOBI-02

**Delivers:**
- Three-zone mobile layout (sticky header, scrollable middle, fixed bottom)
- Budget remaining hero number with color coding
- HSL progress bar (green → yellow → red)
- Responsive design with proper mobile viewport handling

**Success Criteria:**
- [ ] Layout renders correctly on mobile viewport (375px)
- [ ] Progress bar smoothly transitions colors based on budget percentage
- [ ] Hero number color matches progress bar
- [ ] Safe area insets work on iPhone (notch/home indicator)
- [ ] All interactive elements meet 44px minimum touch target

**Depends on:** Phase 1 (database for budget data)

---

## Phase 3: Expense Entry & Management

**Goal:** Complete CRUD flow -- add expenses, view list, delete entries, edit budget.

**Requirements:** BUDG-04, EXPR-01, EXPR-02, EXPR-03, EXPR-04, LIST-01, LIST-02, LIST-03, LIST-04

**Delivers:**
- Quick-entry form (amount, label, person toggle)
- Expense list with current week's entries
- Tap-to-reveal delete on any entry
- Budget amount editable in UI
- Server Actions for all mutations

**Success Criteria:**
- [ ] Can add expense with amount, label, and person selection
- [ ] Numeric keyboard appears on mobile for amount input
- [ ] Person toggle switches between Jason and Shelby
- [ ] Form resets and auto-focuses amount after submit
- [ ] Expense list shows all entries for current week
- [ ] Each entry shows amount, label, person, and timestamp
- [ ] Can delete any entry via tap-to-reveal
- [ ] Budget display updates immediately after add/delete
- [ ] Can change the weekly budget amount from the UI

**Depends on:** Phase 2 (layout to place form and list)

---

## Phase 4: Week Logic & History

**Goal:** Monday auto-reset, rolling week history, and data cleanup.

**Requirements:** WEEK-01, WEEK-02, WEEK-03, WEEK-04

**Delivers:**
- Week boundary calculation (Monday, fixed timezone)
- Current week shown by default
- Navigation to view past weeks (up to 4)
- Automatic cleanup of data older than 4 weeks

**Success Criteria:**
- [ ] New week starts at Monday 12:00 AM in configured timezone
- [ ] Main screen shows current week by default
- [ ] Can navigate to view past 3 weeks
- [ ] Week header shows date range (e.g., "Mar 3 -- Mar 9")
- [ ] Data older than 4 weeks is cleaned up automatically
- [ ] Budget resets to configured amount each new week

**Depends on:** Phase 3 (expenses to organize by week)

---

## Phase 5: PWA & Production Deployment

**Goal:** Installable PWA deployed to Vercel production.

**Requirements:** INFR-04, MOBI-03, MOBI-04, MOBI-05

**Delivers:**
- Web app manifest (manifest.ts)
- Service worker for installability
- App icons (192x192, 512x512)
- Middleware exclusions for PWA files
- Production deployment on Vercel

**Success Criteria:**
- [ ] App is installable on Android Chrome (install prompt appears)
- [ ] App is installable on iOS Safari (Add to Home Screen works)
- [ ] App opens in standalone mode from home screen
- [ ] manifest.webmanifest is accessible and valid
- [ ] Service worker registers successfully in production
- [ ] App is live on Vercel production URL
- [ ] All features work in production (add, delete, budget, weeks)

**Depends on:** Phase 4 (all features complete before deploy)

---

## Phase Summary

| Phase | Goal | Requirements | Est. Plans |
|-------|------|-------------|------------|
| 1 | Scaffolding & Database | INFR-01, INFR-02, INFR-03 | 1 |
| 2 | Core UI & Budget Display | BUDG-01, BUDG-02, BUDG-03, MOBI-01, MOBI-02 | 1-2 |
| 3 | Expense Entry & Management | BUDG-04, EXPR-01-04, LIST-01-04 | 2-3 |
| 4 | Week Logic & History | WEEK-01-04 | 1-2 |
| 5 | PWA & Deployment | INFR-04, MOBI-03-05 | 1-2 |

## Requirement Coverage

All 25 v1 requirements are mapped. 0 unmapped.

---
*Roadmap created: 2026-03-09*
*Last updated: 2026-03-09 after phase 1 planning*
