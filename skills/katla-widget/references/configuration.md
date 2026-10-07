# Katla Widget - Configuration and delivery

The Katla widget collects cookie consent from your visitors and blocks non-consented cookies from being set.

## Prerequisites

- A domain added in Katla
- At least one completed scan

## Installation

Copy the tag from **Install → Widget** in your dashboard and paste it before `</head>`:

```html
<script>(function(){"use strict";var i=document.currentScript,…})();</script>
```

That is the loader itself rather than a link to it - about a kilobyte, and it makes no request
of its own. It runs while the parser is still in `<head>`, which is what gets the Google
Consent Mode default declared before your Google tag can initialise. It then fetches the
engine, which carries your cookie inventory and the banner.

You can also get it from the CLI with `katla install example.com`.

If you would rather link it, `{siteId}.js` serves the same loader:

```html
<script src="https://cdn.katla.app/{siteId}.js" defer></script>
```

This costs a connection to another origin before the loader can run, and on a cold connection
that is usually enough to lose the Consent Mode race. Everything else behaves identically.

### The loader

```html
<script src="https://cdn.katla.app/{siteId}.js" defer></script>
```

Under half a kilobyte. It sends the visitor's browser language and loads only that one - about
15 KB over the wire instead of 19 KB - and any `theme` or `debug` parameter you put on it is
carried across. Resolution happens on our side: an exact match, then the language, then
English, so an unrecognised tag never fails.

Add `locale` to skip detection and pin one language:

```html
<script src="https://cdn.katla.app/{siteId}.js?locale=sv-se" defer></script>
```

An unrecognised locale resolves to the closest language we publish, and to English if there is
no such language.

**If you use Google Consent Mode, use the loader.** It declares the consent default the moment
it is parsed, which is well before your Google tag initialises. The single-language script
declares one too, but from inside a much larger deferred file that `gtag.js` routinely beats -
see [Google Consent Mode](https://docs.katla.app/google-consent-mode).

Winning that race matters only if your Google tag loads before consent at all. It does not have
to, and on katla.app it does not: [basic consent mode](https://docs.katla.app/google-consent-mode) holds the tag back
until the visitor allows analytics, so nothing - not even a cookieless ping carrying their IP
and page URL - reaches Google from someone who declined. Read that section before you decide;
the loader advice above applies either way.

> Linked, the loader needs a request of its own before the cookie guard exists. Inline it
  (below) and that request disappears - the loader is already on the page, and the only fetch
  left is the engine.

### Inlining the loader

The loader is small enough to paste into the page instead of linking to it. Copy it from the
**Install** page in your dashboard, under **Widget → Inline**, and put it in `<head>`:

```html
<script>(function(){"use strict";/* … */})();</script>
```

This is the fastest correct install, and the difference is not marginal. A linked loader cannot
run until a connection to another origin has been opened - DNS, TLS and a round trip, which is
most of its cost, not its size. Inlined, it runs while the parser is still in `<head>`, so the
Google Consent Mode default is declared before your Google tag can initialise. If you have ever
seen `_ga` appear before the banner did, this is the fix.

Pinning a language makes the snippet smaller still, because locale handling drops out of it
entirely:

| Snippet | Brotli |
| --- | --- |
| Automatic language | ~505 B |
| Fixed language | ~482 B |
| Fixed language, self-hosted engine | ~452 B |
| Fixed language, no Consent Mode | ~208 B |

> A pasted copy does not update itself. Re-copy it after changing your Consent Mode setting,
  the theme or the language. Everything else - your cookie inventory, banner appearance, the
  consent records - lives in the engine, which is still fetched at runtime and stays current.

### Self-hosting

Download `{siteId}.consent.js` and serve it from your own origin. It carries every language,
so there is nothing else to fetch and nothing to keep in sync.

If you use Google Consent Mode, take `{siteId}.js` - the loader - as well, and serve the pair
side by side under matching names:

```
/static/katla.js          ← the loader, reference this one
/static/katla.consent.js  ← the engine it pulls in
```

```html
<script src="/static/katla.js" defer></script>
```

The loader resolves its sibling from its own URL - everything before `.js` - so as long as the
two names match it will find your copy rather than reaching back to our CDN. Re-download both
when you change your cookie inventory or widget settings.

Inlining and self-hosting compose, and together they mean no request to us at all. An inlined
loader has no URL of its own, so it cannot resolve a sibling and has to be told where the engine
is: enter its address in **Install → Widget → Inline → Self-hosted engine URL** and it is baked
into the snippet. The file can be called anything.

### Script parameters

You can customize the script behavior with query parameters:

| Parameter | Values | Description |
|-----------|--------|-------------|
| `locale` | `en-GB` | Language for the widget UI |
| `theme` | `light`, `dark` | Override the theme setting |
| `headless` | `true` | Cookie guard only, no visible widget |
| `debug` | `true` | Enable console logging |

```html
<script src="https://cdn.katla.app/{siteId}.js?theme=dark&debug=true"></script>
```

### Content Security Policy

If your site sends a `Content-Security-Policy` header, `script-src` must allow
`https://cdn.katla.app`, and `connect-src` must allow `https://consent.katla.app` so
consent records can be stored. On the free plan, `img-src` must also allow
`https://cdn.katla.app` for the Katla logo in the banner.

```
script-src 'self' https://cdn.katla.app;
connect-src 'self' https://consent.katla.app;
img-src 'self' https://cdn.katla.app;
```

Linked, the loader needs nothing else under a **nonce-based** policy: it copies its own nonce
onto the script it injects. If the injected script is blocked for any other reason, the loader
falls back to the all-languages file, so a visitor is never left without a cookie guard.

Inlined, it is an inline script and `script-src` treats it as one. Under a nonce-based policy,
put your nonce on the tag:

```html
<script nonce="{your-nonce}">(function(){"use strict";…})();</script>
```

One nonce covers both scripts - the loader reads its own and copies it onto the engine.

Under a **hash-based** policy, add the hash of the snippet body to `script-src`, and re-hash it
whenever you re-copy the snippet. If neither fits your policy, link the loader instead: the
`{siteId}.js` tag needs no inline allowance at all.

## How it works

1. The script installs a cookie guard that intercepts `document.cookie` writes.
2. If the visitor hasn't given consent yet, the widget appears.
3. The visitor chooses their preferences (accept all, reject all, or select categories).
4. A `_katla_consent` cookie is stored for 365 days with their choice.
5. Non-consented cookies are blocked and any existing disallowed cookies are removed.

> Functional cookies are always allowed and cannot be rejected.

## Blocking scripts and pixels

The cookie guard stops cookies from being **written**. It does not stop a tracker from loading, and a pixel such as the Meta Pixel reports the visit whether or not it can set a cookie. To hold a tag back until the visitor allows its category, mark it up:

```html
<!-- An external script -->
<script type="text/plain" data-katla-category="marketing" data-katla-src="https://connect.facebook.net/en_US/fbevents.js"></script>

<!-- An inline script -->
<script type="text/plain" data-katla-category="marketing">
  fbq('init', 'YOUR_PIXEL_ID');
  fbq('track', 'PageView');
</script>

<!-- A module script: put its real type in data-katla-type -->
<script type="text/plain" data-katla-type="module" data-katla-category="analytics" data-katla-src="/analytics.js"></script>

<!-- An iframe or image pixel -->
<iframe data-katla-category="marketing" data-katla-src="https://www.youtube.com/embed/VIDEO_ID"></iframe>
<img data-katla-category="marketing" data-katla-src="https://www.facebook.com/tr?id=YOUR_PIXEL_ID&ev=PageView" alt="">
```

`data-katla-category` is one of `analytics`, `marketing`, `personalization` or `security`. When the page loads with that category already allowed, or as soon as the visitor allows it, Katla runs each held-back script in document order. It waits for an external script to load before running the next tag, so a vendor's inline setup can call the library loaded above it.

A tag that has run stays running. If the visitor later narrows their consent, the cookie guard enforces the new choice straight away and the tags stay held back from the next page load.

If your site adds held-back tags after the page has loaded, for example on a client-side route change, call `KatlaConsent.activateBlocked()` to pick them up.

> On WordPress, the [Katla plugin](https://docs.katla.app/wordpress#blocking-scripts) writes this markup for you and runs it itself.

## Customization

You can customize the widget appearance in your site settings. Available options:

| Option | Values | Default |
|--------|--------|---------|
| **Layout** | `box`, `banner`, `modal` | `box` |
| **Position** | `top-left`, `top-right`, `bottom-left`, `bottom-right` | `bottom-right` |
| **Theme** | `system`, `light`, `dark` | `system` |
| **Border radius** | `0` to `10` (shown as `0` to `20px`) | `8` |
| **Colors** | Custom hex values for primary, secondary, accent, background, foreground, muted. Set on the **Appearance** page | Katla violet |
| **Equal buttons** | `true`, `false` | `false` |
| **Google Consent Mode** | `enabled`, `disabled` | `disabled` |
| **Regulation** | `auto`, `gdpr`, `ccpa` | `auto` |
| **DMA Compliant** | `true`, `false` | `false` |
| **Privacy Policy URL** | Any valid URL | - |
| **Cookie Policy URL** | Any valid URL | - |

Both light and dark mode colors can be configured independently.

Colors are your site's branding rather than a banner setting: they live on the site's
**Appearance** page in the dashboard, and the accessibility widget
uses the same ones. After your site's first scan, Katla suggests a palette read from your
homepage there, adjusted so text and buttons stay readable. Nothing changes on your site
until you save it.

### What each color paints

| Color | Where it shows |
|-------|----------------|
| **Primary** | Accept all, the floating cookie icon, switch tracks, focus rings |
| **Secondary** | Reject, unless **Equal buttons** is on |
| **Accent** | The "Always on" badge and the GPC notice |
| **Background** | The card itself, and every grey mixed from it |
| **Foreground** | Headings and body ink, and the scrim behind a blocking layer |
| **Muted** | Descriptions, policy links and cookie table cells |

Everything else - the border, the row divider, the panel behind an open cookie table, the
hover wash, the off state of a switch, and the label color on each button - is derived from
those six. The border in particular has no setting of its own: it is the background and the
text mixed, so a colored card gets a border that belongs to it. The text on a button is
computed from its fill, so a pale secondary gets dark text rather than becoming unreadable.

**Border radius** is a single dial rather than a pixel value. The dashboard shows it as the
corner of the card itself, at twice the stored value, and the widget fans the dial out into
five radii in proportion: the card at 2×, a centered card at 2.25×, a category card at 1.5×,
a button at 1.25× and the small ghost controls at 1×. A dial of 8 - 16px - is the redesign's
own geometry; 0 gives square corners throughout.

### Equal buttons

Reject and Accept all always occupy one row of two equal columns, at the same height,
radius and font weight, in every layout and every language - that is what prominence is
assessed on, and no setting changes it. **Equal buttons** removes the remaining difference
by painting Reject in the primary color too. Turning on **DMA Compliant** implies it.

It applies to that pair only. "Save preferences" in the preference center keeps the
secondary color either way: it commits whatever the visitor set above rather than being the
"no" half of a choice, so painting it primary alongside "Accept all" would leave two equally
loud buttons doing different things.

When **Regulation** is set to `auto`, the widget detects the visitor's region at runtime using timezone and GPC signals. See the [CCPA guide](https://docs.katla.app/ccpa) and [DMA guide](https://docs.katla.app/dma) for details.

When Google Consent Mode is enabled, the widget script automatically signals consent state to Google Analytics and Google Ads tags. See the [Google Consent Mode guide](https://docs.katla.app/google-consent-mode) for details.

## Other embed formats

Besides the JavaScript widget, Katla provides additional endpoints for your site:

| Format | URL | Use case |
|--------|-----|----------|
| **HTML** | `https://cdn.katla.app/{siteId}.html` | Embed as iframe for a standalone cookie settings page |
| **JSON** | `https://cdn.katla.app/{siteId}.json` | Fetch cookie data for custom integrations |
| **Manifest** | `https://cdn.katla.app/{siteId}/manifest.json` | Discover all available resources |
