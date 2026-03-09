# Next.js App Router PWA: Implementation Guide

**Researched:** 2026-03-09
**Overall confidence:** HIGH (based on official Next.js docs + Serwist docs)

## Decision: Which Approach to Use

There are three options. Use this decision tree:

| Need | Approach | Package |
|------|----------|---------|
| Installable + push notifications (no offline) | **Built-in Next.js** | None |
| Installable + offline support + caching | **Serwist** | `@serwist/next` + `serwist` |
| Legacy/existing projects | `@ducanh2912/next-pwa` | Deprecated in favor of Serwist |

**Recommendation for Money-Tracker:** Use the **built-in Next.js approach** for the manifest + service worker registration, and add **Serwist** only if offline support is needed later. Start simple.

The original `next-pwa` (shadowwalker) is unmaintained. `@ducanh2912/next-pwa` is its fork, but the author now recommends Serwist instead. For a money tracker that primarily needs home screen installability, the built-in approach is sufficient.

---

## Option A: Built-in Next.js PWA (Recommended Starting Point)

Source: [Official Next.js PWA Guide](https://nextjs.org/docs/app/guides/progressive-web-apps)

### Step 1: Create the Web App Manifest

Create `app/manifest.ts`:

```ts
import type { MetadataRoute } from 'next'

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'Money Tracker',
    short_name: 'Money',
    description: 'Track your income and expenses',
    start_url: '/',
    display: 'standalone',
    background_color: '#ffffff',
    theme_color: '#000000',
    icons: [
      {
        src: '/icon-192x192.png',
        sizes: '192x192',
        type: 'image/png',
      },
      {
        src: '/icon-512x512.png',
        sizes: '512x512',
        type: 'image/png',
      },
    ],
  }
}
```

Key fields for installability:
- `display: 'standalone'` -- makes it look like a native app (no browser chrome)
- `start_url: '/'` -- where the app opens when launched from home screen
- Icons: you MUST have at least 192x192 and 512x512 PNG icons

### Step 2: Create a Minimal Service Worker

Create `public/sw.js`:

```js
self.addEventListener('push', function (event) {
  if (event.data) {
    const data = event.data.json()
    const options = {
      body: data.body,
      icon: data.icon || '/icon-192x192.png',
      badge: '/icon-192x192.png',
      vibrate: [100, 50, 100],
    }
    event.waitUntil(self.registration.showNotification(data.title, options))
  }
})

self.addEventListener('notificationclick', function (event) {
  event.notification.close()
  event.waitUntil(clients.openWindow('/'))
})
```

### Step 3: Register the Service Worker

In your root layout or a client component:

```tsx
'use client'

import { useEffect } from 'react'

export function ServiceWorkerRegistration() {
  useEffect(() => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/sw.js', {
        scope: '/',
        updateViaCache: 'none',
      })
    }
  }, [])

  return null
}
```

Add `<ServiceWorkerRegistration />` in your root layout.

### Step 4: Security Headers in next.config

```js
// next.config.js or next.config.ts
module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        ],
      },
      {
        source: '/sw.js',
        headers: [
          { key: 'Content-Type', value: 'application/javascript; charset=utf-8' },
          { key: 'Cache-Control', value: 'no-cache, no-store, must-revalidate' },
          { key: 'Content-Security-Policy', value: "default-src 'self'; script-src 'self'" },
        ],
      },
    ]
  },
}
```

### Step 5: Icons

Generate icons at [realfavicongenerator.net](https://realfavicongenerator.net/) and place them in `public/`:
- `icon-192x192.png`
- `icon-512x512.png`

### Step 6: iOS Install Prompt (Optional)

iOS Safari does not support `beforeinstallprompt`. Users must manually "Add to Home Screen" via the share button. You can show a hint:

```tsx
'use client'
import { useState, useEffect } from 'react'

export function InstallPrompt() {
  const [isIOS, setIsIOS] = useState(false)
  const [isStandalone, setIsStandalone] = useState(false)

  useEffect(() => {
    setIsIOS(/iPad|iPhone|iPod/.test(navigator.userAgent) && !(window as any).MSStream)
    setIsStandalone(window.matchMedia('(display-mode: standalone)').matches)
  }, [])

  if (isStandalone || !isIOS) return null

  return (
    <div>
      To install, tap the share button and then "Add to Home Screen".
    </div>
  )
}
```

### That's It for Basic Installability

With just the manifest + icons + HTTPS, modern browsers (Chrome, Edge, Samsung Internet) will show an install prompt. No service worker is strictly required for the install prompt on Android/Chrome, but having one enables push notifications and is good practice.

---

## Option B: Serwist (For Offline Support)

Source: [Serwist Getting Started](https://serwist.pages.dev/docs/next/getting-started)

Use this if you need offline caching, precaching, or runtime caching strategies.

### Install

```bash
npm i @serwist/next && npm i -D serwist
```

### next.config.mjs

```js
import withSerwistInit from "@serwist/next";

const withSerwist = withSerwistInit({
  swSrc: "app/sw.ts",
  swDest: "public/sw.js",
  additionalPrecacheEntries: [{ url: "/~offline", revision: crypto.randomUUID() }],
});

export default withSerwist({
  // your existing next config
});
```

### app/sw.ts

```ts
import { defaultCache } from "@serwist/next/worker";
import type { PrecacheEntry, SerwistGlobalConfig } from "serwist";
import { Serwist } from "serwist";

declare global {
  interface WorkerGlobalScope extends SerwistGlobalConfig {
    __SW_MANIFEST: (PrecacheEntry | string)[] | undefined;
  }
}

declare const self: ServiceWorkerGlobalScope;

const serwist = new Serwist({
  precacheEntries: self.__SW_MANIFEST,
  skipWaiting: true,
  clientsClaim: true,
  navigationPreload: true,
  runtimeCaching: defaultCache,
  fallbacks: {
    entries: [{
      url: "/~offline",
      matcher({ request }) {
        return request.destination === "document";
      },
    }],
  },
});

serwist.addEventListeners();
```

### tsconfig.json additions

```json
{
  "compilerOptions": {
    "types": ["@serwist/next/typings"],
    "lib": ["dom", "dom.iterable", "esnext", "webworker"]
  },
  "exclude": ["public/sw.js"]
}
```

### .gitignore additions

```
public/sw*
public/swe-worker*
```

### Create offline fallback page

Create `app/~offline/page.tsx`:

```tsx
export default function OfflinePage() {
  return (
    <div>
      <h1>You are offline</h1>
      <p>Please check your internet connection and try again.</p>
    </div>
  )
}
```

### Turbopack Warning

Serwist requires Webpack. Next.js 15+ uses Turbopack by default for dev.

```json
{
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build --webpack"
  }
}
```

Use Turbopack in dev (faster), but Webpack for builds (Serwist compatibility).

---

## Critical Gotchas

### 1. Middleware Blocks sw.js and manifest.json

If you use Next.js middleware (e.g., for auth), it WILL intercept requests to `/sw.js` and `/manifest.json` and potentially redirect them. This silently breaks your PWA.

**Fix:** Add early returns in your middleware:

```ts
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Allow PWA files through without any processing
  if (
    request.nextUrl.pathname === '/sw.js' ||
    request.nextUrl.pathname === '/manifest.webmanifest' ||
    request.nextUrl.pathname.startsWith('/icon-')
  ) {
    return NextResponse.next()
  }

  // ... your auth/redirect logic here
}
```

Or use the matcher config to exclude them:

```ts
export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico|sw.js|manifest|icon-).*)'],
}
```

### 2. Service Worker Redirects = Silent Failure

If middleware redirects the `/sw.js` request (e.g., to a login page), the browser silently refuses to register the service worker. No error in console. The PWA just does not work. Always ensure sw.js is served directly, never redirected.

### 3. HTTPS Required

Service workers only work over HTTPS (or localhost). For local testing with push notifications:

```bash
next dev --experimental-https
```

### 4. Cache Hell in Development

If using Serwist or any service worker in dev, old cached content can persist and make it look like your changes are not taking effect. Disable service workers in dev:

```js
// In Serwist config:
disable: process.env.NODE_ENV === "development"
```

For the manual approach, only register the SW in production:

```ts
if ('serviceWorker' in navigator && process.env.NODE_ENV === 'production') {
  navigator.serviceWorker.register('/sw.js')
}
```

### 5. `buildExcludes` for next-pwa Users

If using `@ducanh2912/next-pwa`, you need:

```js
buildExcludes: [/middleware-manifest\.json$/]
```

Without this, the build may fail with App Router.

### 6. iOS Limitations

- No `beforeinstallprompt` event -- cannot programmatically trigger install
- Users must use Share > Add to Home Screen
- Push notifications only work on iOS 16.4+ AND the app must be installed to the home screen first
- Standalone mode works, but some CSS/viewport quirks exist

### 7. Manifest File Naming

When using `app/manifest.ts`, Next.js generates it at `/manifest.webmanifest` (not `/manifest.json`). This is fine -- browsers accept both. But if your middleware or CSP is filtering by filename, use the correct one.

---

## Installability Checklist

For a PWA to show the browser install prompt (Chrome/Edge/Android):

- [ ] Valid web app manifest with `name`, `icons` (192px + 512px), `start_url`, `display: standalone`
- [ ] Served over HTTPS
- [ ] Has a registered service worker (even a minimal no-op one works)
- [ ] User has interacted with the page (not immediate on load)

For iOS Safari: no automatic prompt. User must manually add via Share menu.

---

## Recommended Implementation Order

1. **Add `app/manifest.ts`** -- immediate, zero-risk
2. **Add icons to `public/`** -- 192x192 and 512x512
3. **Add `public/sw.js`** (minimal, just push notification handlers)
4. **Register service worker** in a client component
5. **Add security headers** in next.config
6. **Update middleware** to allow PWA files through
7. **Test installability** on Android Chrome and iOS Safari
8. **(Later) Add Serwist** if offline support is needed

## Sources

- [Official Next.js PWA Guide](https://nextjs.org/docs/app/guides/progressive-web-apps) -- PRIMARY source, updated Feb 2026
- [Serwist Getting Started](https://serwist.pages.dev/docs/next/getting-started) -- for offline support
- [Next.js Manifest API Reference](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/manifest)
- [How to build a Next.js PWA in 2025](https://medium.com/@jakobwgnr/how-to-build-a-next-js-pwa-in-2025-f334cd9755df)
- [Next.js 16 PWA with offline support (LogRocket)](https://blog.logrocket.com/nextjs-16-pwa-offline-support/)
- [PWA setup for Next.js App Router 2026](https://medium.com/@amirjld/how-to-implement-pwa-progressive-web-app-in-next-js-app-router-2026-f25a6797d5e6)
