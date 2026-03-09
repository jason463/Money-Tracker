# Project Research Summary

**Project:** Money Tracker
**Domain:** Mobile-first shared budgeting PWA
**Researched:** 2026-03-09
**Confidence:** HIGH

## Executive Summary

Money Tracker is a dead-simple shared weekly spending tracker for two people (Jason and Shelby), built as a mobile-first PWA on Next.js App Router. Research across four areas confirms this is a straightforward project with well-documented patterns. The core architecture is a three-zone mobile layout (sticky budget header, scrollable expense list, fixed bottom entry form) backed by a Postgres database on Vercel, deployed as an installable PWA with no authentication.

The most significant finding from research is a **database pivot**: the original plan considered Supabase or Vercel Postgres, but Supabase free tier pauses after 7 days of inactivity (breaking the app for a low-traffic household tool), and Vercel Postgres was sunset in mid-2025. **Neon via Vercel Marketplace** is the recommended database. It scales to zero but auto-resumes on the next query (~500ms cold start), so the app never breaks. This changes the data layer from Supabase's client SDK to `@neondatabase/serverless` with tagged template SQL or Drizzle ORM, and eliminates Supabase's realtime subscription feature. Realtime sync between two phones can be handled via polling, `router.refresh()`, or added later if needed.

Key risks are minimal: middleware intercepting PWA files (sw.js, manifest) is the biggest silent-failure gotcha, and using `type="number"` instead of `type="text"` with `inputMode="decimal"` is a common mobile UX mistake. Both are well-documented and easy to prevent.

## Architecture Decision: Database

**Decision: Neon Postgres (via Vercel Marketplace) over Supabase**

| Factor | Neon | Supabase |
|--------|------|----------|
| Inactivity handling | Scale-to-zero, auto-resumes on query (~500ms) | Pauses after 7 days, requires manual unpause |
| Free storage | 512 MB | 500 MB |
| Vercel integration | Native -- env vars auto-populated | Separate account, extra setup |
| Complexity | Just a database | Bundled auth, realtime, storage (unused) |

The Supabase pause behavior is a dealbreaker for a household app with sporadic usage. Neon's auto-resume means the app always works. This means:

- **Drop:** `@supabase/supabase-js`, `@supabase/ssr`, Supabase realtime subscriptions
- **Add:** `@neondatabase/serverless` (or Drizzle ORM with Neon driver)
- **Schema:** Same Postgres schema, just run migrations via Neon's SQL editor or a migration tool
- **Realtime:** Not built-in. For two users, poll on focus/visibility change or use `router.refresh()` after mutations. This is simpler than managing WebSocket subscriptions.

## Key Technical Decisions

### PWA: Built-in Next.js, no third-party package
- Use `app/manifest.ts` (generates `/manifest.webmanifest` automatically)
- Minimal `public/sw.js` for installability
- Register service worker only in production
- Add Serwist later only if offline support is needed
- iOS requires manual "Add to Home Screen" via share menu -- show a hint banner

### Database: Neon via Vercel Marketplace
- `@neondatabase/serverless` for serverless-friendly connections
- Tagged template literals for SQL (`sql\`SELECT * FROM expenses\``)
- Server Actions for mutations (no API route handlers needed)
- `vercel env pull` to get DATABASE_URL locally after Marketplace setup

### UI Layout: Three-zone mobile pattern
- **Sticky header:** Budget remaining (hero number) + progress bar
- **Scrollable middle:** Expense list for current week
- **Fixed bottom footer:** Quick-entry form (amount, label, person toggle, submit)
- Use `min-h-dvh` (not `min-h-screen`) for correct mobile viewport handling
- `pb-[env(safe-area-inset-bottom)]` on footer for iPhone notch/home indicator

### Numeric Input: `type="text"` with `inputMode="decimal"`
- Gets numeric keyboard on mobile without `type="number"` quirks
- Filter non-numeric chars in onChange: `.replace(/[^0-9.]/g, "")`
- Large text (`text-2xl`), right-aligned for currency feel

### Progress Bar: HSL hue rotation
- Hue from 120 (green) to 0 (red) based on percentage spent
- `hsl(${hue}, 80%, 45%)` for smooth continuous color transition
- No discrete color class jumps -- looks polished with minimal code
- `tabular-nums` class on currency amounts to prevent layout shift

### Delete: Tap-to-reveal pattern
- Tap an entry to show a Delete button inline
- Simpler than swipe-to-delete, no library dependencies
- Adequate for 5-15 entries per week with full trust model

### Person Selector: Toggle buttons, not dropdown
- Two large tap targets ("Jason" / "Shelby") side by side
- Active state: `bg-blue-600 text-white`, inactive: `bg-gray-100 text-gray-700`
- 44px minimum touch targets on all interactive elements

## Risks and Gotchas

### Critical

1. **Middleware blocks PWA files** -- If you add Next.js middleware (even for redirects), it intercepts `/sw.js` and `/manifest.webmanifest` requests. The PWA silently fails to install with no console errors. Fix: exclude these paths in middleware matcher config.

2. **RLS/permissions on database** -- With Neon (no Supabase RLS), the database connection string IS the credential. Keep `DATABASE_URL` server-side only (no `NEXT_PUBLIC_` prefix). All database access goes through Server Actions or server components.

3. **Stale data on Vercel** -- Server Components can be statically cached. Use `export const dynamic = 'force-dynamic'` on the main page, or call `cookies()`/`headers()` to opt into dynamic rendering.

### Moderate

4. **Service worker cache hell in development** -- Old cached content persists. Only register the service worker in production (`process.env.NODE_ENV === 'production'`).

5. **FLOAT for money** -- Always use `NUMERIC(10,2)` in Postgres. Floating point causes rounding errors on currency.

6. **Tailwind dynamic class purging** -- Cannot construct classes dynamically (`` `bg-${color}-500` ``). Use inline styles for the HSL progress bar, complete class names for everything else.

### Minor

7. **iOS standalone mode quirks** -- No `beforeinstallprompt` event, must show manual install instructions. Push notifications only on iOS 16.4+ and only when installed.

8. **Week boundary timezone handling** -- "Monday reset" needs a consistent timezone. Use a fixed timezone (e.g., America/Chicago) for `week_start` calculation, not the server's locale.

## Recommended Tech Stack (Final)

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | Next.js 15+ (App Router) | SSR, Server Actions, built-in PWA manifest support |
| Styling | Tailwind CSS v4 | CSS-first config, mobile-first utilities |
| Database | Neon Postgres (Vercel Marketplace) | Auto-resume from zero, free tier sufficient |
| DB Client | `@neondatabase/serverless` | Serverless-optimized, tagged template SQL |
| Deployment | Vercel (CLI) | Native Next.js support, Neon integration |
| PWA | Built-in `manifest.ts` + minimal `sw.js` | No dependencies, installable on Android/iOS |
| ORM | None initially (raw SQL via Neon) | App has 2 tables; ORM is overhead. Add Drizzle if schema grows. |

**Not using:**
- Supabase (pause issue on free tier)
- `@supabase/ssr` (no auth = no need)
- `next-pwa` or Serwist (not needed for basic installability)
- Prisma (heavy for 2 tables)

## Implications for Roadmap

### Phase 1: Project Scaffolding and Database
**Rationale:** Everything depends on the Next.js project structure and database connection.
**Delivers:** Working Next.js app with Neon database, schema created, env vars configured.
**Features:** Database schema (expenses + settings tables), Vercel project linked, development workflow.
**Avoids:** Supabase pause issue by using Neon from the start.

### Phase 2: Core UI and Budget Display
**Rationale:** The three-zone layout is the structural foundation for all features.
**Delivers:** Mobile layout with sticky budget header, progress bar, scrollable list area, fixed footer.
**Features:** Budget remaining display, HSL progress bar, responsive mobile layout.
**Avoids:** `min-h-screen` bug (use `min-h-dvh`), missing safe area insets.

### Phase 3: Expense Entry and List
**Rationale:** The primary user flow -- adding and viewing expenses.
**Delivers:** Quick-entry form, expense list with current week filter, Server Actions for CRUD.
**Features:** Manual expense entry, person toggle, tap-to-delete, week filtering.
**Avoids:** `type="number"` input issues, FLOAT for money, stale Server Component data.

### Phase 4: Week Logic and History
**Rationale:** Week boundaries and auto-reset depend on working expense CRUD.
**Delivers:** Monday auto-reset, rolling 4-week history, week navigation.
**Features:** Auto-reset weekly budget, rolling history, old data cleanup.
**Avoids:** Timezone inconsistency in week boundary calculation.

### Phase 5: PWA and Deployment
**Rationale:** PWA setup is a polish layer; deploy last after features work.
**Delivers:** Installable PWA, production deployment on Vercel, home screen icon.
**Features:** manifest.ts, service worker, icons, middleware exclusions, `vercel --prod`.
**Avoids:** Middleware blocking sw.js, service worker cache issues in dev.

### Phase Ordering Rationale

- Database first because every feature reads/writes data
- UI layout second because it is the container for all interactive components
- Expense CRUD third because it is the core user flow
- Week logic fourth because it layers on top of working expense management
- PWA last because installability is polish, not functionality

### Research Flags

**Phases with standard patterns (skip deeper research):**
- Phase 1 (scaffolding): Well-documented Vercel + Neon setup
- Phase 2 (UI): Standard Tailwind mobile patterns, code samples ready
- Phase 3 (CRUD): Server Actions are well-documented in Next.js docs
- Phase 5 (PWA): Research already contains complete implementation code

**Phase needing attention during planning:**
- Phase 4 (week logic): Timezone handling for week boundaries needs careful design. Research flagged this but did not deeply solve it. Decide on a fixed timezone vs. client timezone during planning.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Official docs for Next.js, Neon, Vercel. Neon transition guide verified. |
| Features | HIGH | Simple CRUD app. All UI patterns verified against MDN + Tailwind docs. |
| Architecture | HIGH | Three-zone layout is a standard mobile pattern. Server Actions well-documented. |
| Pitfalls | HIGH | Middleware/PWA gotcha confirmed across multiple sources. Database pause issue verified. |

**Overall confidence:** HIGH

### Gaps to Address

- **Realtime sync:** Dropping Supabase means no built-in realtime. For two users this is likely fine (refresh on focus), but if both are logging expenses simultaneously, decide whether to add polling or accept occasional stale reads.
- **Week boundary timezone:** Need to pick a fixed timezone for Monday reset. Research identified the issue but the specific timezone (likely America/Chicago or similar) should be decided during Phase 4 planning.
- **Data cleanup strategy:** "Rolling 4 weeks" needs a mechanism -- cron job via Vercel Cron, or cleanup on page load. Not deeply researched.

## Sources

### Primary (HIGH confidence)
- [Next.js PWA Guide](https://nextjs.org/docs/app/guides/progressive-web-apps) -- manifest.ts, service worker setup
- [Neon Vercel Marketplace](https://vercel.com/marketplace/neon) -- database provisioning
- [Neon transition guide](https://neon.com/docs/guides/vercel-postgres-transition-guide) -- Vercel Postgres sunset
- [Vercel CLI docs](https://vercel.com/docs/cli) -- deployment workflow
- [Supabase docs](https://supabase.com/docs) -- schema patterns (applicable to raw Postgres)
- [MDN inputMode](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/inputmode) -- numeric keyboard
- [Tailwind CSS docs](https://tailwindcss.com/docs) -- mobile-first patterns, v4 theming

### Secondary (MEDIUM confidence)
- [Vercel App Router mistakes blog](https://vercel.com/blog/common-mistakes-with-the-next-js-app-router-and-how-to-fix-them) -- caching gotchas
- [Neon vs Supabase comparison](https://hrekov.com/blog/vercel-vs-supabase-database-comparison) -- free tier details
- [Serwist docs](https://serwist.pages.dev/docs/next/getting-started) -- offline support (deferred)

---
*Research completed: 2026-03-09*
*Ready for roadmap: yes*
