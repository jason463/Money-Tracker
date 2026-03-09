# Phase 1: Project Scaffolding & Database - Research

**Researched:** 2026-03-09
**Domain:** Next.js project setup, Neon Postgres database, Tailwind CSS v4
**Confidence:** HIGH

## Summary

Phase 1 bootstraps a greenfield Next.js project with a Neon Postgres database. This is a well-documented path: `create-next-app` scaffolds the project, Neon via Vercel Marketplace provides the database, and `@neondatabase/serverless` connects them. The project has zero existing code.

The most important technical detail: Next.js is now at version 16.x, but the project roadmap specifies Next.js 15. Use `npx create-next-app@15` to pin to version 15 (latest 15.x patch). This avoids Next.js 16 breaking changes (async request APIs fully required, middleware renamed to proxy) while using a stable, well-documented version.

The database schema is minimal: two tables (expenses and settings). Use `NUMERIC(10,2)` for money, never FLOAT. The `@neondatabase/serverless` driver (v1.0.0+) uses tagged template literals for safe SQL -- no ORM needed for two tables.

**Primary recommendation:** Scaffold with `npx create-next-app@15`, provision Neon via Vercel Marketplace, create schema via Neon SQL Editor, pull env vars with `vercel env pull`.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| INFR-01 | Next.js App Router project scaffolded with Tailwind CSS | `create-next-app@15` with `--tailwind` flag; Tailwind v4 uses CSS-first config with `@import "tailwindcss"` |
| INFR-02 | Neon Postgres database provisioned with expenses and settings tables | Neon via Vercel Marketplace; schema via SQL Editor; `NUMERIC(10,2)` for money |
| INFR-03 | Database connection working from Server Actions and server components | `@neondatabase/serverless` with tagged template SQL; `neon()` function for HTTP queries |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Next.js | 15.x (latest patch) | App framework | Roadmap specifies v15; pin with `create-next-app@15` |
| React | 19.x | UI library | Bundled with Next.js 15 |
| Tailwind CSS | v4 | Styling | CSS-first config, no tailwind.config.js needed |
| @tailwindcss/postcss | v4 | PostCSS plugin | Required for Tailwind v4 in Next.js |
| @neondatabase/serverless | ^1.0.0 | Database driver | Serverless-optimized, tagged template SQL, HTTP queries |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| typescript | ^5.x | Type safety | Included by create-next-app |
| postcss | ^8.x | CSS processing | Required by Tailwind v4 |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Raw SQL via @neondatabase/serverless | Drizzle ORM | ORM adds overhead for 2 tables; add later if schema grows |
| @neondatabase/serverless HTTP | Pool/Client (WebSocket) | HTTP is faster for one-shot queries; WebSocket only needed for interactive transactions |

**Installation:**
```bash
npx create-next-app@15 . --typescript --tailwind --eslint --app --turbopack --import-alias "@/*" --yes
npm install @neondatabase/serverless
```

## Architecture Patterns

### Recommended Project Structure
```
src/
├── app/
│   ├── layout.tsx          # Root layout
│   ├── page.tsx            # Main page (server component)
│   └── globals.css         # Tailwind CSS imports
├── lib/
│   └── db.ts               # Database connection utility
```

### Pattern 1: Database Utility Module
**What:** Single module exporting a configured `sql` tagged template function.
**When to use:** Every server component and server action that queries the database.
**Example:**
```typescript
// src/lib/db.ts
import { neon } from '@neondatabase/serverless';

export const sql = neon(process.env.DATABASE_URL!);
```

Usage in a server component:
```typescript
// src/app/page.tsx
import { sql } from '@/lib/db';

export const dynamic = 'force-dynamic';

export default async function Home() {
  const expenses = await sql`SELECT * FROM expenses ORDER BY created_at DESC`;
  const [settings] = await sql`SELECT * FROM settings WHERE key = 'weekly_budget'`;

  return (
    <div>
      <p>Budget: ${settings?.value}</p>
      <ul>
        {expenses.map((e: any) => (
          <li key={e.id}>{e.label}: ${e.amount}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Pattern 2: Server Actions for Mutations
**What:** Use `"use server"` functions for database writes.
**When to use:** Any insert, update, or delete operation.
**Example:**
```typescript
// src/app/actions.ts
'use server';

import { sql } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function addExpense(formData: FormData) {
  const amount = formData.get('amount') as string;
  const label = formData.get('label') as string;
  const person = formData.get('person') as string;

  await sql`
    INSERT INTO expenses (amount, label, person)
    VALUES (${amount}, ${label}, ${person})
  `;

  revalidatePath('/');
}
```

### Pattern 3: Force Dynamic Rendering
**What:** Opt out of static caching for pages that read from the database.
**When to use:** Any page displaying live data.
**Example:**
```typescript
// At the top of any page.tsx that queries the database
export const dynamic = 'force-dynamic';
```

### Anti-Patterns to Avoid
- **Exposing DATABASE_URL to the client:** Never use `NEXT_PUBLIC_` prefix for database credentials. All DB access goes through server components or server actions.
- **Using `neon()` as a regular function call:** In v1.0.0+, `sql(...)` with parentheses throws an error. Always use tagged template literals: `` sql`...` ``.
- **Creating `neon()` inside request handlers:** Create the `sql` function once in a module, not per-request. The HTTP driver is stateless; module-level initialization is safe and correct.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| SQL parameterization | Manual string escaping | Tagged template literals via `neon()` | Built-in SQL injection protection |
| Project scaffolding | Manual file creation | `create-next-app@15` | Correct TypeScript, ESLint, Tailwind config |
| Environment variable management | Manual .env files | `vercel env pull` | Syncs with Vercel dashboard, keeps secrets out of git |
| Database provisioning | Manual Postgres install | Neon via Vercel Marketplace | Auto-resume from zero, env vars auto-populated |

**Key insight:** The entire scaffolding and database setup is a well-paved path. Every component has an official CLI or wizard. Do not manually configure what the tools configure for you.

## Common Pitfalls

### Pitfall 1: Using FLOAT for Money
**What goes wrong:** Rounding errors on currency calculations (e.g., 0.1 + 0.2 != 0.3).
**Why it happens:** IEEE 754 floating point cannot represent all decimal fractions exactly.
**How to avoid:** Use `NUMERIC(10,2)` in Postgres. Store amounts as exact decimal values.
**Warning signs:** Amounts displaying as 10.000000001 or budget math being off by fractions of a cent.

### Pitfall 2: Stale Data from Server Component Caching
**What goes wrong:** Database changes don't appear on page reload.
**Why it happens:** Next.js can statically cache server component output by default.
**How to avoid:** Add `export const dynamic = 'force-dynamic'` to pages that read live data, or call `cookies()`/`headers()` to opt into dynamic rendering.
**Warning signs:** Data appears stale after mutations even though the database shows updated values.

### Pitfall 3: Tagged Template vs Function Call (Neon v1.0.0+)
**What goes wrong:** Runtime error: "This function can now be called only as a tagged-template function."
**Why it happens:** `@neondatabase/serverless` v1.0.0 removed support for calling the query function with parentheses.
**How to avoid:** Always use `` sql`SELECT ...` `` (tagged template), never `sql("SELECT ...")` (function call). For manual parameterization, use `sql.query('SELECT ... WHERE id = $1', [id])`.
**Warning signs:** Runtime errors mentioning "tagged-template function."

### Pitfall 4: Missing `vercel link` Before `vercel env pull`
**What goes wrong:** `vercel env pull` fails or pulls wrong variables.
**Why it happens:** The CLI doesn't know which Vercel project to pull from.
**How to avoid:** Run `vercel link` first to associate the local directory with the Vercel project, then `vercel env pull .env.local`.
**Warning signs:** Error messages about unlinked project, or empty `.env.local` file.

### Pitfall 5: Tailwind v4 CSS Import Syntax
**What goes wrong:** Tailwind classes don't apply; no styles render.
**Why it happens:** Using old `@tailwind base; @tailwind components; @tailwind utilities;` directives instead of v4 syntax.
**How to avoid:** In `globals.css`, use `@import "tailwindcss";` (v4 syntax). `create-next-app@15` with `--tailwind` should set this up correctly, but verify.
**Warning signs:** Tailwind utility classes have no effect; page renders unstyled.

## Code Examples

### Database Schema (run in Neon SQL Editor)
```sql
-- Expenses table
CREATE TABLE expenses (
  id SERIAL PRIMARY KEY,
  amount NUMERIC(10, 2) NOT NULL,
  label TEXT NOT NULL,
  person TEXT NOT NULL CHECK (person IN ('Jason', 'Shelby')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Settings table with default budget
CREATE TABLE settings (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

-- Insert default weekly budget
INSERT INTO settings (key, value) VALUES ('weekly_budget', '200.00');
```

### Database Utility Module
```typescript
// src/lib/db.ts
import { neon } from '@neondatabase/serverless';

if (!process.env.DATABASE_URL) {
  throw new Error('DATABASE_URL environment variable is required');
}

export const sql = neon(process.env.DATABASE_URL);
```

### Querying from a Server Component
```typescript
// src/app/page.tsx
import { sql } from '@/lib/db';

export const dynamic = 'force-dynamic';

export default async function Home() {
  const expenses = await sql`
    SELECT id, amount, label, person, created_at
    FROM expenses
    ORDER BY created_at DESC
  `;

  const [budget] = await sql`
    SELECT value FROM settings WHERE key = 'weekly_budget'
  `;

  return (
    <main>
      <h1>Money Tracker</h1>
      <p>Weekly Budget: ${budget?.value ?? '0.00'}</p>
      <p>Expenses: {expenses.length}</p>
    </main>
  );
}
```

### Environment Variables (.env.local via vercel env pull)
```
# Populated by `vercel env pull .env.local`
DATABASE_URL="postgres://username:password@ep-xxx.us-east-2.aws.neon.tech/neondb?sslmode=require"
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `@vercel/postgres` SDK | `@neondatabase/serverless` | Mid-2025 | Vercel Postgres sunset; use Neon driver directly |
| `tailwind.config.js` + directives | CSS-first config (`@import "tailwindcss"`) | Jan 2025 (Tailwind v4) | No config file needed; simpler setup |
| `neon()` as function or tagged template | Tagged template only | `@neondatabase/serverless` v1.0.0 | Function-style calls throw errors |
| `npx create-next-app@latest` = Next.js 15 | `@latest` = Next.js 16 | Early 2026 | Pin with `@15` to match roadmap |

**Deprecated/outdated:**
- `@vercel/postgres`: Sunset. Use `@neondatabase/serverless` instead.
- `tailwind.config.js`: Optional in v4. CSS-first config is the standard approach.
- `min-h-screen`: Use `min-h-dvh` for correct mobile viewport (relevant for later phases).

## Open Questions

1. **Next.js 15 vs 16**
   - What we know: Roadmap specifies Next.js 15. Current `@latest` is 16.x. Next.js 15 is still maintained.
   - What's unclear: Whether the user specifically wants v15 or just wrote it when v15 was latest.
   - Recommendation: Use `create-next-app@15` as the roadmap specifies. This avoids v16 breaking changes and is simpler for a small project.

2. **Neon Database Provisioning Method**
   - What we know: Can use Vercel Marketplace (auto-populates env vars) or Neon Console directly.
   - What's unclear: Whether the user already has a Neon account or Vercel project set up.
   - Recommendation: Use Vercel Marketplace flow (integrates env vars automatically). If not possible in automated task, provide manual Neon Console instructions as fallback.

3. **Schema Migration Strategy**
   - What we know: Two tables, no ORM, raw SQL.
   - What's unclear: Whether to use a migration tool or just run SQL directly.
   - Recommendation: For 2 tables, run SQL directly in Neon SQL Editor. No migration tool needed. Store the schema SQL in `src/lib/schema.sql` for reference.

## Sources

### Primary (HIGH confidence)
- [Next.js create-next-app docs](https://nextjs.org/docs/app/api-reference/cli/create-next-app) - CLI flags and defaults
- [Next.js App Router installation](https://nextjs.org/docs/app/getting-started/installation) - project setup
- [Neon serverless driver docs](https://neon.com/docs/serverless/serverless-driver) - tagged template SQL, HTTP usage
- [@neondatabase/serverless npm](https://www.npmjs.com/package/@neondatabase/serverless) - v1.0.0 breaking changes
- [Neon Vercel Marketplace](https://vercel.com/marketplace/neon) - database provisioning
- [Neon Postgres NUMERIC docs](https://neon.com/docs/data-types/decimal) - NUMERIC(10,2) for money
- [Tailwind CSS v4 Next.js guide](https://tailwindcss.com/docs/guides/nextjs) - CSS-first installation

### Secondary (MEDIUM confidence)
- [Neon Vercel Postgres transition guide](https://neon.com/docs/guides/vercel-postgres-transition-guide) - migration from @vercel/postgres
- [Next.js v15 to v16 upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16) - breaking changes in v16

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Official docs for all components verified
- Architecture: HIGH - Standard Next.js App Router patterns, well-documented
- Pitfalls: HIGH - Known issues confirmed across multiple sources (NUMERIC for money, tagged template breaking change, caching)

**Research date:** 2026-03-09
**Valid until:** 2026-04-09 (stable technologies, 30-day window)
