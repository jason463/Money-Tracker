# UI Patterns Research: Mobile-First Money Tracker

**Researched:** 2026-03-09
**Confidence:** HIGH (patterns verified against official docs and multiple sources)

---

## 1. Mobile-First Tailwind Foundations

### Touch Targets

Apple requires 44x44pt minimum. Google Material uses 48dp. WCAG 2.5.8 (AA) requires 24px minimum, AAA requires 44px. Use 44px as the floor for all interactive elements.

```tsx
// Button base — 44px min touch target
<button className="min-h-[44px] min-w-[44px] px-4 py-3 text-base">
  Add Entry
</button>

// Icon button — pad to 44px even if icon is 24px
<button className="flex items-center justify-center h-11 w-11">
  <TrashIcon className="h-5 w-5" />
</button>
```

### Spacing and Typography

Mobile-first means default styles target phones, `md:` prefix for tablets/desktop.

```
Font sizes:
  - Body text:      text-base (16px) — minimum for mobile readability
  - Budget amount:  text-3xl or text-4xl — the hero number, glanceable
  - Entry amounts:  text-lg (18px)
  - Labels/meta:    text-sm (14px) — only for secondary info

Spacing:
  - Card padding:   p-4 (16px)
  - Between cards:  space-y-3 (12px)
  - Form fields:    space-y-4 (16px gap between fields)
  - Page padding:   px-4 (16px horizontal)

Safe area (PWA on iPhone):
  - Add pb-[env(safe-area-inset-bottom)] to bottom-fixed elements
  - Add <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

### Key Layout Pattern

Full viewport mobile layout with sticky header showing budget status:

```tsx
<div className="flex flex-col min-h-dvh bg-gray-50">
  {/* Sticky budget display */}
  <header className="sticky top-0 z-10 bg-white border-b px-4 py-3 shadow-sm">
    <BudgetDisplay />
  </header>

  {/* Scrollable entry list */}
  <main className="flex-1 overflow-y-auto px-4 py-3">
    <EntryList />
  </main>

  {/* Fixed bottom entry form or FAB */}
  <footer className="sticky bottom-0 bg-white border-t px-4 py-3 pb-[env(safe-area-inset-bottom)]">
    <QuickEntryForm />
  </footer>
</div>
```

Use `min-h-dvh` (dynamic viewport height) instead of `min-h-screen` to handle mobile browser chrome correctly.

---

## 2. Budget Progress Bar with Color Transitions

### Recommended: Smooth HSL Gradient

For a budget tracker, "percentage spent" maps naturally to a hue rotation: green (lots left) through yellow (caution) to red (over budget).

```tsx
function BudgetBar({ spent, budget }: { spent: number; budget: number }) {
  const pct = Math.min((spent / budget) * 100, 100);
  // HSL hue: 120 (green) -> 60 (yellow) -> 0 (red)
  const hue = Math.max(0, ((100 - pct) / 100) * 120);

  return (
    <div className="space-y-2">
      {/* Remaining amount — colored to match bar */}
      <div className="flex justify-between items-baseline">
        <span
          className="text-3xl font-bold tabular-nums"
          style={{ color: `hsl(${hue}, 75%, 35%)` }}
        >
          ${(budget - spent).toFixed(2)}
        </span>
        <span className="text-sm text-gray-500">
          of ${budget.toFixed(0)} left
        </span>
      </div>

      {/* Progress bar */}
      <div className="h-3 w-full bg-gray-200 rounded-full overflow-hidden">
        <div
          className="h-full rounded-full transition-all duration-500 ease-out"
          style={{
            width: `${pct}%`,
            backgroundColor: `hsl(${hue}, 80%, 45%)`,
          }}
        />
      </div>
    </div>
  );
}
```

Why HSL over discrete classes: The transition is smooth — at 40% spent the bar is naturally yellow-green, at 70% it is orange. No awkward jumps between color bands. The `transition-all duration-500` handles animated width + color changes together.

### Alternative: Discrete Color Steps (simpler, less polished)

```tsx
function getBarColor(pct: number) {
  if (pct < 50) return "bg-green-500";
  if (pct < 75) return "bg-yellow-500";
  if (pct < 90) return "bg-orange-500";
  return "bg-red-500";
}

function getRemainingColor(pct: number) {
  if (pct < 50) return "text-green-700";
  if (pct < 75) return "text-yellow-700";
  if (pct < 90) return "text-orange-700";
  return "text-red-700";
}
```

Note: Tailwind classes cannot be dynamically constructed (e.g. `` `bg-${color}-500` `` will NOT work — Tailwind purges unused classes at build time). Always use complete class names.

---

## 3. Quick-Entry Form for Phone Use

### Key Principles

1. Use `inputMode="decimal"` with `type="text"` (NOT `type="number"`) to get the numeric keyboard with a decimal point.
2. Large tap targets for the person selector (Jason/Shelby toggle).
3. Minimal fields: amount, label, who. Three fields max.
4. Auto-focus the amount field on form open.

```tsx
function QuickEntryForm({ onSubmit }: { onSubmit: (entry: Entry) => void }) {
  const [amount, setAmount] = useState("");
  const [label, setLabel] = useState("");
  const [who, setWho] = useState<"Jason" | "Shelby">("Jason");
  const amountRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const parsed = parseFloat(amount);
    if (!parsed || parsed <= 0) return;
    onSubmit({ amount: parsed, label: label.trim() || "Misc", who });
    setAmount("");
    setLabel("");
    amountRef.current?.focus();
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-3">
      <div className="flex gap-3">
        {/* Amount — numeric keyboard, large text */}
        <input
          ref={amountRef}
          type="text"
          inputMode="decimal"
          placeholder="0.00"
          value={amount}
          onChange={(e) => setAmount(e.target.value.replace(/[^0-9.]/g, ""))}
          className="flex-1 min-h-[44px] px-3 text-2xl font-bold text-right
                     border rounded-lg focus:ring-2 focus:ring-blue-500
                     focus:outline-none"
          autoComplete="off"
        />

        {/* Label — optional */}
        <input
          type="text"
          placeholder="What for?"
          value={label}
          onChange={(e) => setLabel(e.target.value)}
          className="flex-1 min-h-[44px] px-3 text-base
                     border rounded-lg focus:ring-2 focus:ring-blue-500
                     focus:outline-none"
        />
      </div>

      <div className="flex gap-3">
        {/* Person toggle — large tap targets */}
        {(["Jason", "Shelby"] as const).map((name) => (
          <button
            key={name}
            type="button"
            onClick={() => setWho(name)}
            className={`flex-1 min-h-[44px] rounded-lg font-medium text-base
              transition-colors ${
                who === name
                  ? "bg-blue-600 text-white"
                  : "bg-gray-100 text-gray-700"
              }`}
          >
            {name}
          </button>
        ))}

        {/* Submit */}
        <button
          type="submit"
          className="min-h-[44px] px-6 bg-green-600 text-white
                     rounded-lg font-semibold text-base
                     active:bg-green-700 transition-colors"
        >
          Add
        </button>
      </div>
    </form>
  );
}
```

### Why `inputMode="decimal"` over `type="number"`

- `type="number"` has inconsistent stepper UI across browsers, prevents text selection, and has quirks with `.value` returning empty string for invalid input.
- `inputMode="decimal"` gives the numeric keyboard on mobile without any of those problems.
- On iOS it shows digits + decimal point. On Android it shows digits + decimal + minus.
- Filter non-numeric characters in the onChange handler.

---

## 4. Delete Pattern: Tap-to-Delete (Recommended over Swipe)

### Why NOT Swipe-to-Delete for This App

Swipe-to-delete adds complexity (touch tracking, thresholds, animations, library dependencies) for a feature that will be used rarely. This is a simple tracker with maybe 5-15 entries per week and full trust between two users. A simpler pattern wins.

### Recommended: Tap to Reveal Delete Button

```tsx
function EntryItem({
  entry,
  onDelete,
}: {
  entry: Entry;
  onDelete: (id: string) => void;
}) {
  const [showDelete, setShowDelete] = useState(false);

  return (
    <div
      className="flex items-center justify-between px-4 py-3 bg-white
                 rounded-lg border min-h-[44px] active:bg-gray-50
                 transition-colors"
      onClick={() => setShowDelete(!showDelete)}
    >
      <div className="flex-1 min-w-0">
        <div className="flex justify-between items-baseline">
          <span className="text-base font-medium truncate">
            {entry.label}
          </span>
          <span className="text-lg font-bold tabular-nums ml-3">
            ${entry.amount.toFixed(2)}
          </span>
        </div>
        <span className="text-sm text-gray-500">{entry.who}</span>
      </div>

      {showDelete && (
        <button
          onClick={(e) => {
            e.stopPropagation();
            onDelete(entry.id);
          }}
          className="ml-3 min-h-[44px] min-w-[44px] flex items-center
                     justify-center bg-red-500 text-white rounded-lg
                     text-sm font-medium active:bg-red-600
                     transition-colors"
        >
          Delete
        </button>
      )}
    </div>
  );
}
```

### Alternative: If You Do Want Swipe Later

Use `react-swipe-to-delete-ios` (0 dependencies, iOS-style UX). But start with tap-to-delete and validate whether swipe is actually needed.

---

## 5. Color Theming: Keep It Simple

For a two-person money tracker with no dark mode requirement, skip theming infrastructure entirely. Use Tailwind's default color palette directly.

### Recommended Palette

```
Primary action (Add button, focused inputs):  blue-600
Person A (Jason):                               blue-500
Person B (Shelby):                              purple-500
Destructive (delete):                           red-500
Budget remaining text/bar:                      Dynamic HSL (see section 2)
Background:                                     gray-50
Card background:                                white
Text primary:                                   gray-900
Text secondary:                                 gray-500
Borders:                                        gray-200
```

### If You Want Theme Variables Later (Tailwind v4)

Tailwind v4 uses CSS-first `@theme` directive — no JS config file needed:

```css
/* app/globals.css */
@import "tailwindcss";

@theme {
  --color-primary: #2563eb;     /* blue-600 */
  --color-jason: #3b82f6;       /* blue-500 */
  --color-shelby: #a855f7;      /* purple-500 */
  --color-danger: #ef4444;      /* red-500 */
}
```

Then use `bg-primary`, `text-jason`, etc. in markup. But for this app's simplicity, hardcoded Tailwind colors are fine. Add theming only if you find yourself changing colors in multiple places.

---

## 6. PWA Setup (Bonus — Relevant to Mobile-First)

Next.js App Router has built-in PWA manifest support. No `next-pwa` package needed.

```tsx
// app/manifest.ts
import type { MetadataRoute } from "next";

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: "Money Tracker",
    short_name: "Tracker",
    start_url: "/",
    display: "standalone",
    background_color: "#f9fafb",
    theme_color: "#2563eb",
    icons: [
      { src: "/icon-192.png", sizes: "192x192", type: "image/png" },
      { src: "/icon-512.png", sizes: "512x512", type: "image/png" },
    ],
  };
}
```

Add to layout.tsx `<head>`:
```tsx
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="default" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
```

---

## Sources

- [Tailwind CSS Responsive Design](https://tailwindcss.com/docs/responsive-design) — mobile-first breakpoint system
- [WCAG 2.5.8 Target Size Guide](https://www.allaccessible.org/blog/wcag-258-target-size-minimum-implementation-guide) — touch target requirements
- [MDN inputmode](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/inputmode) — numeric keyboard on mobile
- [React Aria Number Field](https://react-spectrum.adobe.com/blog/how-we-internationalized-our-numberfield.html) — why `type="text"` + `inputMode` beats `type="number"`
- [Tailwind CSS Theme Variables](https://tailwindcss.com/docs/theme) — v4 CSS-first theming
- [Next.js PWA Guide](https://nextjs.org/docs/app/guides/progressive-web-apps) — built-in manifest support
- [Building Swipeable List with React](https://malcoded.com/posts/react-swipeable-list/) — touch event handling from scratch
