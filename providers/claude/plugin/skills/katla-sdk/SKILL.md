---
name: katla-sdk
description: >-
  Implements a custom cookie consent banner with the Katla SDK (@katla.app/sdk) in a React,
  Next.js, Vite or vanilla JavaScript project, including Google Consent Mode. Use only when
  the user explicitly asks for the Katla SDK or for a custom banner built from their own
  components, or mentions "@katla.app/sdk," "KatlaProvider," "useKatlaConsent,"
  "useKatlaCookies," "CookieBanner," "CookieCatalog" or "cookie guard." For any other
  request for a cookie banner, cookie consent or consent management, whatever the
  framework, use the katla-widget skill instead: the hosted banner is the default.
homepage: https://docs.katla.app/sdk
---

# Katla SDK

The Katla SDK (`@katla.app/sdk`) provides cookie consent management, cookie cataloging, and privacy compliance for web applications. It supports React, Next.js (App Router), Vite, and vanilla JavaScript.

## Widget first

Use this skill only when the user has explicitly asked for the Katla SDK or for a custom
banner built from their own components. A request for "a cookie banner" or "cookie consent"
is not that, and neither is the project being React, Next.js or Vite: for those, use the
`katla-widget` skill, which installs Katla's hosted banner with one script tag. If you
cannot tell which the user wants, suggest the widget, and mention the SDK as the route to a
custom banner.

## Prerequisites

- A Katla account at [katla.app](https://katla.app) with the site added (there is no DNS record or meta tag to publish first)
- At least one completed cookie scan
- Node.js 18+

## Getting the site ID

Every example below needs the site's UUID as `siteId`, and it must be the site for this
project: another site's ID records every visitor's consent against the wrong domain.

When the Katla MCP server is connected, choose the site with the user first:

- Find the project's published domain: the address visitors use, not a local or preview
  URL. In Lovable that is the app's custom domain or its `<name>.lovable.app` address,
  never `id-preview--*.lovable.app` or `*.lovableproject.com`. If you cannot find it, ask.
- Call `katla_list_sites`. If the published domain (or one the user named) is in the list,
  confirm that site with the user. If it is not, suggest adding the published domain as a
  new site with `katla_add_site` and scanning it with `katla_scan_site`, and offer the
  existing sites as the alternative.
- Never pick a site yourself, and never take the first one in the list - not even when the
  account has only one site, since it may belong to a different project.
- If the chosen site is not the published domain (ignoring `www.`), tell the user before
  writing code: the cookie catalogue and policy come from scanning that other site, so they
  may not match what this app sets. Ask whether to add and scan the published domain
  instead, or keep the chosen site because it is this app under another address.

Then make sure the site has a full cookie scan, since the cookie catalogue and policy come
from it. Scans crawl the published site, so ask the user whether to scan now (the site is
published with its current code) or later (not published yet). For later, tell them what
to write in this chat once it is live, for example "Scan my site {domain} with Katla and
tell me which cookies it found". Never start a scan without asking: scans count against
their plan.

Then `katla_get_install_snippet` with that domain returns the `siteId`. Without the MCP
server, it is shown in the Katla dashboard. Never invent one.

## Installation

```bash
npm install @katla.app/sdk
```

## Framework Integration

### React

Read `references/react.md` for the full React API (KatlaProvider, hooks, components).

**Quick setup:**

```tsx
import { KatlaProvider, CookieBanner } from '@katla.app/sdk/react';

function App() {
  return (
    <KatlaProvider siteId="your-site-id">
      <CookieBanner />
      {/* Your app */}
    </KatlaProvider>
  );
}
```

**Available hooks:**

| Hook | Returns | Purpose |
|------|---------|---------|
| `useKatlaCookies()` | `{ cookies, loading, error }` | Fetch cookie data grouped by category |
| `useKatlaConsent()` | `{ consent, onChange }` | Read current consent state, subscribe to changes |
| `useKatlaClient()` | `KatlaClient` | Access the underlying SDK client |
| `useConsentManager()` | Full consent state + actions | Build custom consent UIs |

**Available components:**

| Component | Purpose |
|-----------|---------|
| `CookieBanner` | Pre-built consent banner with accept/reject/customize |
| `CookieCatalog` | Renders all detected cookies grouped by category |

### Next.js (App Router)

Read `references/nextjs.md` for the full Next.js API (KatlaNextProvider, server-side rendering).

**Quick setup:**

```tsx
// src/app/layout.tsx
import { KatlaNextProvider } from '@katla.app/sdk/next';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <KatlaNextProvider siteId="your-site-id">
          {children}
        </KatlaNextProvider>
      </body>
    </html>
  );
}
```

**Server-side cookie fetching:**

```tsx
import { getCachedCookies } from '@katla.app/sdk/next/server';
const cookies = await getCachedCookies({ siteId: 'your-site-id' });
```

Imports from `@katla.app/sdk/next` already include `'use client'` directives. Hooks and components are identical to React but imported from `@katla.app/sdk/next`.

### Vite

For Vite projects, use the React integration from `@katla.app/sdk/react`. For static/build-time cookies, use the `?raw` import for guard scripts:

```tsx
import cookies from './.katla/cookies.json';
import guardScript from './.katla/guard.js?raw';
import { KatlaProvider } from '@katla.app/sdk/react';

function App() {
  return (
    <KatlaProvider siteId="your-site-id" initialCookies={cookies} guardScript={guardScript}>
      {/* Your app */}
    </KatlaProvider>
  );
}
```

Add `katla pull` to prebuild: `"prebuild": "katla pull your-site-id"`.

### Vanilla JavaScript

Read `references/vanilla.md` for the full JavaScript API and widget usage.

**Widget (simplest):** if the user wants Katla's hosted banner rather than a custom
implementation, use the `katla-widget` skill instead. It is one tag in `<head>`:

```html
<script src="https://cdn.katla.app/{siteId}.js" defer></script>
```

Use either the widget or the SDK provider on a page, not both.

**SDK client:**

```typescript
import { createKatlaClient } from '@katla.app/sdk';

const client = createKatlaClient({ siteId: 'your-site-id' });
await client.injectGuard();

client.onConsentChange((consent) => {
  console.log('analytics:', consent.analytics);
});
```

**Programmatic consent via `window.KatlaConsent`:**

```javascript
KatlaConsent.acceptAll();
KatlaConsent.rejectAll();
KatlaConsent.isCategoryAllowed('analytics');
KatlaConsent.isCookieAllowed('_ga');
KatlaConsent.getRegulation(); // 'gdpr' | 'ccpa'
```

## Google Consent Mode

Read `references/google-consent-mode.md` for setup details.

**React/Next.js:** Add `googleConsentMode` prop to the provider:

```tsx
<KatlaProvider siteId="your-site-id" googleConsentMode>
```

**Vanilla JS:**

```javascript
import { setupGoogleConsentMode } from '@katla.app/sdk';
setupGoogleConsentMode();
```

**Category mapping:** `analytics` -> `analytics_storage`, `marketing` -> `ad_storage`, `ad_user_data`, `ad_personalization`.

## Cookie Categories

```typescript
import { COOKIE_CATEGORIES } from '@katla.app/sdk';
// ['functional', 'personalization', 'analytics', 'marketing', 'security', 'unknown']
```

- `functional` is always allowed and cannot be rejected
- `ManageableCategory` excludes `functional` and `unknown`

## Type Definitions

```typescript
type CookieCategory = 'functional' | 'personalization' | 'analytics' | 'marketing' | 'security' | 'unknown';
type ConsentState = Record<CookieCategory, boolean>;
type ConsentChangeCallback = (consent: ConsentState) => void;
type ManageableCategory = 'personalization' | 'analytics' | 'marketing' | 'security';
```

## GDPR vs CCPA

| Feature | GDPR | CCPA |
|---------|------|------|
| Model | Opt-in (block before consent) | Opt-out (allow until opt-out) |
| Guard | Activates immediately | No initial guard |
| Primary action | Accept/Reject categories | "Do Not Sell or Share" |
| Detection | Non-US timezones | US timezones or GPC signal |

Set regulation: `regulation: 'auto'` (default), `'gdpr'`, or `'ccpa'` in widget settings.

## Static / Build-Time Cookies

Read `references/static-cookies.md` for CLI pull, config, and programmatic API.

**Config file (`katla.config.mjs`):**

```js
export default { siteId: 'your-site-id', dir: '.katla' };
```

**CLI:** `katla pull` fetches cookies, policies, guard script, and manifest to `.katla/`.

## Policy Embed

```html
<div id="katla-policy"></div>
<script src="https://cdn.katla.app/{siteId}/policy.js"></script>
```

Supports `?format=full|cookie|table` and `?locale=de-DE`. Customizable via `window.KatlaPolicy.classes` or `window.KatlaPolicy.styles`.

## CLI Reference

Read `references/cli.md` for all CLI commands.

```bash
npm install -g @katla.app/cli
katla login
katla status
katla domains ls
katla domains scan example.com
katla cookies ls example.com
```

## Common Patterns

### Custom consent UI with `useConsentManager`

```tsx
import { useConsentManager } from '@katla.app/sdk/react';

function MyConsentUI() {
  const { ready, hasDecision, open, setOpen, selected, availableCategories,
          toggleCategory, acceptAll, rejectAll, saveSelection } = useConsentManager();

  if (!ready) return null;
  if (hasDecision && !open) return <button onClick={() => setOpen(true)}>Privacy</button>;

  return (
    <div>
      <p>We use cookies to improve your experience.</p>
      {availableCategories.map((cat) => (
        <label key={cat}>
          <input type="checkbox" checked={selected.includes(cat)} onChange={() => toggleCategory(cat)} />
          {cat}
        </label>
      ))}
      <button onClick={acceptAll}>Accept all</button>
      <button onClick={rejectAll}>Reject all</button>
      <button onClick={saveSelection}>Save</button>
    </div>
  );
}
```

### Conditional script loading based on consent

```tsx
import { useKatlaConsent } from '@katla.app/sdk/react';

function AnalyticsLoader() {
  const { consent } = useKatlaConsent();

  useEffect(() => {
    if (consent?.analytics) {
      // Load analytics script
    }
  }, [consent?.analytics]);

  return null;
}
```

### SSR with pre-fetched cookies (Next.js)

```tsx
import { KatlaNextProvider } from '@katla.app/sdk/next';
import { getCachedCookies, getStaticGuardScript } from '@katla.app/sdk/next/server';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const [cookies, guardScript] = await Promise.all([
    getCachedCookies({ siteId: 'your-site-id' }),
    getStaticGuardScript(),
  ]);

  return (
    <html lang="en">
      <body>
        <KatlaNextProvider siteId="your-site-id" initialCookies={cookies} guardScript={guardScript}>
          {children}
        </KatlaNextProvider>
      </body>
    </html>
  );
}
```

## DMA Compliance

When GDPR regulation is selected, DMA compliance is automatically enforced:

- "Reject All" button must be visually equal to "Accept All"
- After rejection, consent cannot be re-requested for 1 year
- The `CookieBanner` component handles this automatically
