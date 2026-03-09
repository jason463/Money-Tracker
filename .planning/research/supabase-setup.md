# Supabase + Next.js App Router Setup (No Auth)

**Researched:** 2026-03-09
**Confidence:** HIGH (official docs verified)

---

## 1. Packages to Install

```bash
npm install @supabase/supabase-js @supabase/ssr
```

- `@supabase/supabase-js` -- core Supabase client
- `@supabase/ssr` -- cookie-aware wrappers for SSR frameworks (Next.js App Router)

Since this app has **no auth**, the `@supabase/ssr` package is optional. You can skip it and use `@supabase/supabase-js` directly with `createClient`. The SSR package adds cookie-based session management which is only needed for auth flows. For a no-auth app, a single shared browser client is simpler.

**Recommendation:** Use `@supabase/supabase-js` only. Skip `@supabase/ssr` -- it adds complexity for zero benefit when there is no auth.

## 2. Environment Variables

Create `.env.local`:

```
NEXT_PUBLIC_SUPABASE_URL=https://your-project-id.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIs...
```

Both are prefixed with `NEXT_PUBLIC_` so they are available in client components. This is safe because the anon key is designed to be public -- RLS policies control actual data access.

Get these from: Supabase Dashboard > Project Settings > API.

Note: Supabase is transitioning to calling the anon key "publishable key" (`NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`). Both names work. Use whichever you prefer; the legacy `ANON_KEY` name is still widely used.

## 3. Client Setup

### Simplest approach (no auth = no SSR client needed)

Create one file: `lib/supabase.ts`

```typescript
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

This singleton client works in both server components (for initial data fetch) and client components (for realtime subscriptions and mutations).

### Server Components -- Data Fetching

```typescript
// app/page.tsx (server component)
import { supabase } from '@/lib/supabase'

export default async function Page() {
  const { data: expenses } = await supabase
    .from('expenses')
    .select('*')
    .order('created_at', { ascending: false })

  return <ExpenseList initialExpenses={expenses ?? []} />
}
```

Server components fetch data at request time. No `useEffect`, no loading spinners for initial load.

### Client Components -- Mutations and Realtime

```typescript
'use client'
import { supabase } from '@/lib/supabase'

// Use supabase directly for inserts, updates, deletes, and realtime subscriptions
```

**Why this works without `@supabase/ssr`:** The SSR package exists to synchronize auth sessions between server and browser via cookies. With no auth, there is no session to synchronize. The base `createClient` works fine in both environments for simple data operations.

## 4. Database Schema

### SQL to run in Supabase SQL Editor:

```sql
-- Expenses table
CREATE TABLE expenses (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  amount NUMERIC(10,2) NOT NULL,
  label TEXT NOT NULL,
  person TEXT NOT NULL CHECK (person IN ('person1', 'person2')),
  created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
  week_start DATE NOT NULL
);

-- Settings table (single row)
CREATE TABLE settings (
  id INTEGER PRIMARY KEY DEFAULT 1 CHECK (id = 1),  -- enforces single row
  budget_amount NUMERIC(10,2) NOT NULL DEFAULT 150.00
);

-- Insert default settings row
INSERT INTO settings (budget_amount) VALUES (150.00);

-- Index for querying expenses by week
CREATE INDEX idx_expenses_week_start ON expenses(week_start);
```

**Key design decisions:**
- `id` uses UUID (Supabase convention, avoids sequential ID guessing)
- `person` uses a CHECK constraint rather than a foreign key to a users table (no auth = no users table)
- `settings` uses `CHECK (id = 1)` to enforce exactly one row -- simple and effective
- `week_start` is a DATE for easy week-based grouping/filtering
- `amount` uses NUMERIC(10,2) for precise currency values (never use FLOAT for money)

## 5. Row-Level Security (RLS) for Public Access

Since there is no auth, you need RLS policies that allow the `anon` role full access. **Do NOT disable RLS** -- that is a bad habit. Instead, enable RLS and create explicit permissive policies.

```sql
-- EXPENSES: full public CRUD
ALTER TABLE expenses ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Allow public select on expenses"
  ON expenses FOR SELECT TO anon USING (true);

CREATE POLICY "Allow public insert on expenses"
  ON expenses FOR INSERT TO anon WITH CHECK (true);

CREATE POLICY "Allow public update on expenses"
  ON expenses FOR UPDATE TO anon USING (true) WITH CHECK (true);

CREATE POLICY "Allow public delete on expenses"
  ON expenses FOR DELETE TO anon USING (true);

-- SETTINGS: public read + update only (no insert/delete)
ALTER TABLE settings ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Allow public select on settings"
  ON settings FOR SELECT TO anon USING (true);

CREATE POLICY "Allow public update on settings"
  ON settings FOR UPDATE TO anon USING (true) WITH CHECK (true);
```

**Security note:** This makes the data fully public. Anyone with the Supabase URL and anon key can read/write. This is acceptable for a small shared household app. If you later want to restrict access, add Supabase Auth and tighten the policies.

## 6. Enable Realtime

Realtime is NOT enabled by default on tables. You must add them to the `supabase_realtime` publication:

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE expenses;
ALTER PUBLICATION supabase_realtime ADD TABLE settings;
```

Run this in the SQL Editor. Alternatively, toggle it in: Dashboard > Database > Publications > supabase_realtime.

## 7. Realtime Subscription Pattern

Realtime subscriptions **only work in client components** (they use WebSockets which require the browser).

### Pattern: Client component with initial server data + realtime updates

```typescript
'use client'

import { useEffect, useState } from 'react'
import { supabase } from '@/lib/supabase'

type Expense = {
  id: string
  amount: number
  label: string
  person: string
  created_at: string
  week_start: string
}

export function ExpenseList({ initialExpenses }: { initialExpenses: Expense[] }) {
  const [expenses, setExpenses] = useState<Expense[]>(initialExpenses)

  useEffect(() => {
    const channel = supabase
      .channel('expenses-changes')
      .on(
        'postgres_changes',
        { event: '*', schema: 'public', table: 'expenses' },
        (payload) => {
          if (payload.eventType === 'INSERT') {
            setExpenses((prev) => [payload.new as Expense, ...prev])
          } else if (payload.eventType === 'DELETE') {
            setExpenses((prev) => prev.filter((e) => e.id !== payload.old.id))
          } else if (payload.eventType === 'UPDATE') {
            setExpenses((prev) =>
              prev.map((e) => (e.id === payload.new.id ? (payload.new as Expense) : e))
            )
          }
        }
      )
      .subscribe()

    // Cleanup on unmount
    return () => {
      supabase.removeChannel(channel)
    }
  }, [])

  return (/* render expenses */)
}
```

### Key realtime details:
- Channel name (`'expenses-changes'`) can be any string except `'realtime'`
- `event: '*'` listens to INSERT, UPDATE, and DELETE
- Always clean up with `supabase.removeChannel(channel)` in the useEffect return
- `payload.new` contains the new row data; `payload.old` contains the old row data (for DELETE, only `old` is populated by default)
- To get `payload.old` on UPDATE/DELETE with full row data, run: `ALTER TABLE expenses REPLICA IDENTITY FULL;`

### Settings subscription (simpler -- updates only):

```typescript
useEffect(() => {
  const channel = supabase
    .channel('settings-changes')
    .on(
      'postgres_changes',
      { event: 'UPDATE', schema: 'public', table: 'settings' },
      (payload) => {
        setBudget(payload.new.budget_amount)
      }
    )
    .subscribe()

  return () => { supabase.removeChannel(channel) }
}, [])
```

## 8. Common Mutations

### Add expense:
```typescript
const { error } = await supabase.from('expenses').insert({
  amount: 12.50,
  label: 'Groceries',
  person: 'person1',
  week_start: '2026-03-09',  // computed: Monday of current week
})
```

### Delete expense:
```typescript
const { error } = await supabase.from('expenses').delete().eq('id', expenseId)
```

### Update budget:
```typescript
const { error } = await supabase
  .from('settings')
  .update({ budget_amount: 200.00 })
  .eq('id', 1)
```

## 9. Pitfalls and Gotchas

| Pitfall | Prevention |
|---------|------------|
| Empty data returned from queries | RLS is enabled by default on dashboard-created tables. If you forget the `TO anon` policies, all queries return empty arrays with no error. |
| Realtime not firing | You MUST add tables to the `supabase_realtime` publication. This is a separate step from creating the table. |
| `payload.old` is empty on DELETE | By default, Supabase only sends the row ID in `old`. Run `ALTER TABLE expenses REPLICA IDENTITY FULL;` to get all columns. |
| Using FLOAT for money | Always use `NUMERIC(10,2)`. Floating point causes rounding errors (e.g., 0.1 + 0.2 !== 0.3). |
| Stale data after realtime event | The realtime payload contains the changed row. Update local state directly from the payload rather than re-fetching. |
| Multiple subscriptions on re-render | Always return the cleanup function from `useEffect`. Missing cleanup = duplicate listeners = duplicate UI updates. |
| Supabase client in server components with `@supabase/ssr` | For a no-auth app, this adds unnecessary complexity. Use `@supabase/supabase-js` directly. |

## 10. Complete File Structure

```
lib/
  supabase.ts           # Single createClient export
app/
  page.tsx              # Server component: fetch initial data, pass to client
  components/
    ExpenseList.tsx     # 'use client': renders expenses, subscribes to realtime
    AddExpenseForm.tsx  # 'use client': form to insert expenses
    BudgetDisplay.tsx   # 'use client': shows budget, subscribes to settings changes
.env.local              # NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY
```

## Sources

- [Supabase Next.js Quickstart](https://supabase.com/docs/guides/getting-started/quickstarts/nextjs)
- [Supabase Realtime with Next.js](https://supabase.com/docs/guides/realtime/realtime-with-nextjs)
- [Supabase Postgres Changes](https://supabase.com/docs/guides/realtime/postgres-changes)
- [Supabase Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase API Keys](https://supabase.com/docs/guides/api/api-keys)
- [Creating a Supabase Client for SSR](https://supabase.com/docs/guides/auth/server-side/creating-a-client)
- [Supabase SSR package discussion](https://github.com/orgs/supabase/discussions/28997)
