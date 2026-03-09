# Requirements: Money Tracker

**Defined:** 2026-03-09
**Core Value:** Jason and Shelby can see how much of their weekly discretionary budget remains at a glance, with zero friction to log a purchase.

## v1 Requirements

### Infrastructure

- [ ] **INFR-01**: Next.js App Router project scaffolded with Tailwind CSS
- [ ] **INFR-02**: Neon Postgres database provisioned with expenses and settings tables
- [ ] **INFR-03**: Database connection working from Server Actions and server components
- [ ] **INFR-04**: Project deployed to Vercel via CLI

### Budget Display

- [ ] **BUDG-01**: Weekly budget remaining displayed as prominent hero number
- [ ] **BUDG-02**: Progress bar shows percentage of budget spent with smooth HSL color transition (green → yellow → red)
- [ ] **BUDG-03**: Remaining budget text color matches progress bar color
- [ ] **BUDG-04**: Budget amount is editable in the UI by either person

### Expense Entry

- [ ] **EXPR-01**: User can add expense with amount, label, and person (Jason or Shelby)
- [ ] **EXPR-02**: Amount input uses numeric keyboard on mobile (`inputMode="decimal"`)
- [ ] **EXPR-03**: Person selector is a two-button toggle (Jason / Shelby), not a dropdown
- [ ] **EXPR-04**: Form resets and auto-focuses amount after successful submission

### Expense List

- [ ] **LIST-01**: Current week's expenses displayed in scrollable list
- [ ] **LIST-02**: Each entry shows amount, label, person, and time
- [ ] **LIST-03**: Any entry can be deleted by either person (tap-to-reveal delete)
- [ ] **LIST-04**: Deleting an entry updates the budget display immediately

### Week Logic

- [ ] **WEEK-01**: New week starts every Monday (fixed timezone)
- [ ] **WEEK-02**: Current week's data shown by default on the main screen
- [ ] **WEEK-03**: Rolling history of ~4 weeks viewable
- [ ] **WEEK-04**: Data older than 4 weeks is automatically cleaned up

### Mobile & PWA

- [ ] **MOBI-01**: Mobile-first responsive layout (three-zone: sticky header, scrollable list, fixed bottom form)
- [ ] **MOBI-02**: All interactive elements have 44px minimum touch targets
- [ ] **MOBI-03**: App is installable as PWA on Android and iOS home screens
- [ ] **MOBI-04**: Web app manifest with app name, icons (192x192, 512x512), and theme color
- [ ] **MOBI-05**: Service worker registered for PWA installability

## v2 Requirements

### Polish

- **POLSH-01**: Offline support via Serwist (queue entries when offline, sync when online)
- **POLSH-02**: iOS install hint banner ("Add to Home Screen" instructions)
- **POLSH-03**: Realtime sync between devices (polling on focus/visibility change)
- **POLSH-04**: Push notifications when budget hits warning threshold

## Out of Scope

| Feature | Reason |
|---------|--------|
| Bank connections / transaction imports | Manual-only by design |
| Categories, tags, or analytics | Simple tracker, not a budgeting tool |
| Recurring bills, subscriptions, rent | Discretionary spending only |
| Budgeting by category | One shared pot, no sub-budgets |
| User accounts or authentication | Shared URL, full trust model |
| Multiple budgets or households | Two people, one budget |
| Configurable user names | Hardcoded Jason and Shelby |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFR-01 | Phase 1 | Pending |
| INFR-02 | Phase 1 | Pending |
| INFR-03 | Phase 1 | Pending |
| INFR-04 | Phase 5 | Pending |
| BUDG-01 | Phase 2 | Pending |
| BUDG-02 | Phase 2 | Pending |
| BUDG-03 | Phase 2 | Pending |
| BUDG-04 | Phase 3 | Pending |
| EXPR-01 | Phase 3 | Pending |
| EXPR-02 | Phase 3 | Pending |
| EXPR-03 | Phase 3 | Pending |
| EXPR-04 | Phase 3 | Pending |
| LIST-01 | Phase 3 | Pending |
| LIST-02 | Phase 3 | Pending |
| LIST-03 | Phase 3 | Pending |
| LIST-04 | Phase 3 | Pending |
| WEEK-01 | Phase 4 | Pending |
| WEEK-02 | Phase 4 | Pending |
| WEEK-03 | Phase 4 | Pending |
| WEEK-04 | Phase 4 | Pending |
| MOBI-01 | Phase 2 | Pending |
| MOBI-02 | Phase 2 | Pending |
| MOBI-03 | Phase 5 | Pending |
| MOBI-04 | Phase 5 | Pending |
| MOBI-05 | Phase 5 | Pending |

**Coverage:**
- v1 requirements: 25 total
- Mapped to phases: 25
- Unmapped: 0

---
*Requirements defined: 2026-03-09*
*Last updated: 2026-03-09 after initial definition*
