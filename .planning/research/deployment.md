# Deployment Research: Vercel + Database for Money Tracker

**Researched:** 2026-03-09
**Overall confidence:** HIGH

---

## 1. Vercel CLI Setup & Deployment

### Initial Setup

```bash
npm install -g vercel
vercel login          # Opens browser for auth
```

### Deploy Commands

```bash
vercel                # Deploy to preview (prompts to link project on first run)
vercel --prod         # Deploy to production
```

On first `vercel` run, the CLI auto-detects Next.js and configures build settings. No manual config needed for App Router projects.

### Workflow

1. `vercel link` -- connects local directory to Vercel project
2. `vercel env pull` -- downloads env vars to `.env.local` for local dev
3. `vercel` -- deploy preview
4. `vercel --prod` -- deploy production

The CLI creates a `.vercel/` directory locally (add to `.gitignore`).

**Confidence:** HIGH -- sourced from [Vercel CLI docs](https://vercel.com/docs/cli) and [Vercel deploy docs](https://vercel.com/docs/cli/deploy).

---

## 2. Database: Neon (via Vercel Marketplace) -- NOT Supabase

### Recommendation: Use Neon directly through Vercel Marketplace

**Vercel Postgres was sunset in mid-2025.** It transitioned to Neon's native integration. The current path is: Vercel Marketplace -> Neon Postgres.

### Why Neon over Supabase for this project

| Factor | Neon (via Vercel) | Supabase |
|--------|-------------------|----------|
| **Pausing** | Scale-to-zero (auto-resumes on query) | **Pauses after 7 days of inactivity, requires manual unpause** |
| **Free storage** | 512 MB per project | 500 MB |
| **Free compute** | 100 CU-hours/month (enough for light use) | Shared CPU, always-on when not paused |
| **Auth/extras** | None (just a database) | Bundled auth, realtime, storage (unused overhead) |
| **Vercel integration** | Native -- env vars auto-populated | Marketplace integration available but extra step |
| **Active projects** | Up to 20 | 2 on free tier |
| **Billing** | Unified through Vercel | Separate account |

**The decisive factor is pausing.** Supabase free tier pauses your database after 7 days of inactivity. For a tiny app used by two people, there will be weeks where nobody logs a purchase. The database would pause and the app would break until someone manually unpauses it in the Supabase dashboard. Workarounds exist (GitHub Actions cron pings) but they are hacky.

Neon uses scale-to-zero instead: the compute shuts down after 5 minutes idle but **auto-resumes on the next query** with a ~500ms cold start. The app just works, always.

### Neon Setup via Vercel Marketplace

1. Go to Vercel Dashboard -> Storage -> Browse Marketplace -> Neon
2. Create a Neon database (auto-provisions account if needed)
3. Environment variables (`DATABASE_URL`, `POSTGRES_URL`, etc.) are auto-added to your Vercel project
4. Run `vercel env pull` to get them locally

### Connecting from Next.js

Use `@neondatabase/serverless` for serverless-friendly connections, or use any standard Postgres client (Drizzle, Prisma, raw `pg`). For this simple app, Drizzle ORM or even raw SQL via `@neondatabase/serverless` is fine.

```bash
npm install @neondatabase/serverless
```

```typescript
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL!);
const result = await sql`SELECT * FROM expenses WHERE week_start = ${weekStart}`;
```

**Confidence:** HIGH -- sourced from [Neon Vercel transition guide](https://neon.com/docs/guides/vercel-postgres-transition-guide), [Neon plans](https://neon.com/docs/introduction/plans), [Vercel Marketplace](https://vercel.com/marketplace/neon).

---

## 3. Environment Variable Management

### The Setup

| Where | File | Purpose |
|-------|------|---------|
| Local dev | `.env.local` | Database URL, any secrets for local development |
| Vercel | Dashboard or CLI | Production/preview env vars (encrypted at rest) |

### Workflow

```bash
# Add vars via CLI
vercel env add DATABASE_URL production
vercel env add DATABASE_URL preview
vercel env add DATABASE_URL development

# Or just set them in Vercel Dashboard -> Settings -> Environment Variables

# Pull to local
vercel env pull    # Creates .env.local with Development env vars
```

### Key Rules

- **Never commit `.env.local`** -- already in default Next.js `.gitignore`
- `NEXT_PUBLIC_` prefix exposes vars to the browser -- do NOT use this for database URLs
- When using Neon via Vercel Marketplace, `DATABASE_URL` and related vars are auto-populated in Vercel -- just `vercel env pull` to get them locally
- Env var changes only apply to NEW deployments, not retroactively
- 64 KB total limit across all env vars per deployment (plenty for this app)

**Confidence:** HIGH -- sourced from [Vercel env docs](https://vercel.com/docs/environment-variables).

---

## 4. Custom Domain (Optional)

### Setup Steps

1. **Add domain in Vercel Dashboard:** Project Settings -> Domains -> Add
2. **Configure DNS at your registrar:**
   - Root domain (`example.com`): A record -> `76.76.21.21`
   - Subdomain (`www.example.com`): CNAME -> `cname.vercel-dns.com`
3. **Verify** in Vercel Dashboard (propagation: minutes to 24 hours)
4. **SSL auto-provisioned** via Let's Encrypt -- no action needed

### CLI Alternative

```bash
vercel domains add yourdomain.com
# Or after deploying:
vercel alias set <deployment-url> yourdomain.com
```

### Notes

- Custom domains are free on all Vercel plans
- Add both root and `www` -- Vercel auto-redirects one to the other
- If using Cloudflare: set DNS to "DNS only" (gray cloud), not "Proxied"
- For this project, the default `*.vercel.app` URL is probably fine to start -- add custom domain later if wanted

**Confidence:** HIGH -- sourced from [Vercel domains docs](https://vercel.com/docs/domains/set-up-custom-domain).

---

## 5. Vercel + App Router Gotchas

### Caching Behavior (CRITICAL)

Next.js App Router defaults changed between versions. In recent Next.js (15+), `fetch` requests are **not cached by default** (unlike Next.js 14 where they were). This is actually better for a dynamic spending tracker -- data should always be fresh.

However, if using Server Components that read from the database, make sure routes are dynamic:

```typescript
// In your page.tsx
export const dynamic = 'force-dynamic';
// OR use cookies()/headers() which auto-opt into dynamic rendering
```

### Server Actions vs Route Handlers

For mutations (adding/deleting expenses, updating budget), use **Server Actions** directly from Client Components instead of creating API Route Handlers. Simpler code, fewer files.

```typescript
// actions.ts
'use server'
export async function addExpense(formData: FormData) {
  // Insert into database directly
}
```

### Suspense Boundaries

Place `<Suspense>` wrappers **above** async Server Components, not inside them:

```tsx
// Correct
<Suspense fallback={<Loading />}>
  <ExpenseList />   {/* This is the async component */}
</Suspense>
```

### Edge Runtime Middleware Issues

Some developers report middleware failing on Edge Runtime in recent versions. If using middleware (probably not needed for this app), test on deployment or use `export const runtime = 'nodejs'`.

### Security: Keep Next.js Updated

CVEs were disclosed in 2025 affecting App Router (DoS via crafted requests, source code exposure of Server Actions). Vercel blocks vulnerable versions and has WAF rules, but keep Next.js updated.

### Build Cache Issues

If deployment shows stale data, set `VERCEL_FORCE_NO_BUILD_CACHE=1` as an env var. Usually not needed but good to know.

**Confidence:** HIGH -- sourced from [Vercel's common App Router mistakes blog](https://vercel.com/blog/common-mistakes-with-the-next-js-app-router-and-how-to-fix-them).

---

## 6. Recommended Deployment Checklist for Money Tracker

```
1. [ ] Create GitHub repo, push code
2. [ ] `npm install -g vercel && vercel login`
3. [ ] `vercel link` (connect to new Vercel project)
4. [ ] Add Neon Postgres via Vercel Marketplace (auto-populates DATABASE_URL)
5. [ ] `vercel env pull` to get DATABASE_URL locally
6. [ ] Run database migrations (create tables)
7. [ ] `vercel --prod` to deploy
8. [ ] Test on phone, add to home screen (PWA)
9. [ ] (Optional) Add custom domain in Vercel Dashboard
```

---

## Sources

- [Vercel CLI docs](https://vercel.com/docs/cli)
- [Vercel deploy command](https://vercel.com/docs/cli/deploy)
- [Next.js on Vercel](https://vercel.com/docs/frameworks/full-stack/nextjs)
- [Neon Vercel transition guide](https://neon.com/docs/guides/vercel-postgres-transition-guide)
- [Neon plans & limits](https://neon.com/docs/introduction/plans)
- [Neon on Vercel Marketplace](https://vercel.com/marketplace/neon)
- [Vercel environment variables](https://vercel.com/docs/environment-variables)
- [Vercel custom domains](https://vercel.com/docs/domains/set-up-custom-domain)
- [Common App Router mistakes](https://vercel.com/blog/common-mistakes-with-the-next-js-app-router-and-how-to-fix-them)
- [Neon vs Supabase free tier comparison](https://hrekov.com/blog/vercel-vs-supabase-database-comparison)
- [Supabase pause prevention](https://github.com/travisvn/supabase-pause-prevention)
