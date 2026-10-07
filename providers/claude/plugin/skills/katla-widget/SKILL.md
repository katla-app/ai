---
name: katla-widget
description: >-
  Installs Katla's hosted cookie consent widget (one script tag) on a website and wires the
  rest of the site to it. Use when the user wants a cookie banner, cookie consent, GDPR/CCPA
  consent or a cookie policy on their site, or mentions "Katla widget," "katla install,"
  "KatlaConsent," "katla.open," "policy embed," or "cookie settings link." This is the
  default for every site: Vite, React, Next.js, plain HTML, Lovable, site builders. Only
  when the user explicitly asks for the Katla SDK or for a custom banner built in code, use
  the katla-sdk skill instead.
homepage: https://docs.katla.app/widget
---

# Katla Widget

The widget is Katla's hosted consent banner. One script tag installs a cookie guard that
refuses non-consented cookies at `document.cookie`, shows the banner to new visitors, stores
their choice, and records it as consent evidence. The banner's look, copy, regulation and
Google Consent Mode setting all live in the Katla dashboard, not in the site's code.

## Widget or SDK?

| The user wants | Use |
|---|---|
| A working, compliant banner with no UI work | This skill (the widget) |
| The banner restyled (colors, layout, position, radius, theme) | This skill - it is dashboard configuration, not code. Colors are the site's branding, shared with the accessibility widget |
| The Katla SDK by name, or a custom banner built from their own components | The `katla-sdk` skill |

Always suggest the widget. Move to the SDK only when the user explicitly asks for it or for
a custom banner: a React or Next.js project is not a reason on its own, and neither is a
plain request for a cookie banner or cookie consent. Use one or the other on a page, never
both.

## Prerequisites

1. **Find the project's published domain.** This is the address visitors use, and the one
   Katla has to scan. In Lovable it is the app's custom domain if one is connected,
   otherwise its `<name>.lovable.app` address. Look for it in what you already know about
   the project (the published URL, a canonical or `og:url` tag in `index.html`, the
   conversation). Editor and preview addresses (`id-preview--*.lovable.app`,
   `*.lovableproject.com`, localhost) are not it. If you cannot find it, ask the user, and
   if the app has never been published, ask them to publish it first.
2. **Choose the site with the user.** The tag is tied to one Katla site, and installing
   another site's tag records every visitor's consent against the wrong domain. With the
   Katla MCP server, call `katla_list_sites`, then:
   - If the published domain (or a domain the user named) is in the list, confirm that
     site with the user before using it.
   - If it is not, show the user the domains in the account and suggest adding the
     published domain as a new site and scanning it. Offer the existing sites as the
     alternative, not the default.
   - Never pick a site yourself, and never take the first one in the list - not even when
     the account has only one site, since it may belong to a different project.
   - To add one, `katla_add_site` with the published domain. There is no DNS record or
     meta tag to publish first.
3. **Check the site matches what is published.** If the site the user chose is not the
   published domain (ignoring a leading `www.`), stop and tell them before installing:
   the banner lists the cookies Katla found by scanning *that* site, and the cookie policy
   is written from the same scan, so both may not match what this app sets. Then ask what
   to do:
   - add the published domain as a new site and scan it (usually right), or
   - keep the chosen site, if it really is this app under another address (for example
     the custom domain of the same Lovable project) - then scan it if its last scan is
     older than the app's latest publish.
4. **A full cookie scan.** The banner lists, and the policy is written from, the cookies the
   last scan found, so a site that has never been scanned shows an empty cookie list. Scans
   crawl the *published* site, never a local or preview URL. Check with `katla_get_site`
   whether the site has a completed scan that is newer than its latest publish, and if not,
   ask the user:
   - **Scan now** - if the site is published with its current code. `katla_scan_site`,
     then poll `katla_get_scan_status` until it completes, and tell them what was found.
     This scan cannot see the Katla tag, which is not installed yet: Step 7 asks for one
     more once the install is published.
   - **Scan later** - if the site is not published yet, or the changes being made now (new
     analytics, embeds) are not live yet. Carry on with the install: the scan Step 7 ends
     with, once the user has published, is then this one too.
   Never scan a site on the user's behalf without asking: scans count against their plan.

## Step 1: Get the tag

Call `katla_get_install_snippet` with the site's domain. It returns the `siteId` and the
script tag:

```html
<script async src="https://cdn.katla.app/{siteId}.js"></script>
```

Use the tag exactly as returned. Never guess or invent a site ID.

Without the MCP server, the tag is in the Katla dashboard under **Install → Widget**, or from
the CLI with `katla install example.com`.

## Step 2: Put it in `<head>`

The tag goes in the document `<head>`, before any analytics, ads or other tag that sets
cookies. The guard can only refuse cookies that are written after it has run.

| Stack | Where |
|---|---|
| Vite, Lovable, plain React SPA | `index.html`, inside `<head>`, above other scripts |
| Plain HTML / multi-page site | Every page's `<head>`, or the shared head partial |
| Next.js App Router | `app/layout.tsx`: a plain `<script>` inside `<head>` in the root layout |
| Next.js Pages Router | `pages/_document.tsx`, inside `<Head>` |
| WordPress | Use the Katla WordPress plugin instead of pasting the tag |
| Site builders | The site-wide "custom code in head" setting |

Vite / Lovable example (`index.html`):

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <script async src="https://cdn.katla.app/{siteId}.js"></script>
    <!-- analytics and other tags go below -->
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

Do not load the widget from a React component or `useEffect`: by then other scripts may
already have set cookies.

## Step 3: Gate your own tags on consent

The guard blocks cookies, but a script the site loads itself should also wait for consent.
The widget exposes `window.KatlaConsent`:

```javascript
function loadAnalytics() {
  /* inject the analytics script here */
}

function onConsent() {
  if (window.KatlaConsent?.isCategoryAllowed('analytics')) loadAnalytics();
}

onConsent();
window.KatlaConsent?.subscribe(onConsent);
```

Categories: `functional` (always allowed), `personalization`, `analytics`, `marketing`,
`security`, `unknown`. See `references/javascript-api.md` for the full API.

For Google Analytics or Google Ads, prefer switching on **Google Consent Mode** in the site
settings (`katla_update_site_settings` with `googleConsentMode: true`) over gating gtag by
hand. The Katla tag must still come before the Google tag.

## Step 4: Add a "Cookie settings" link

Visitors need a route back to their choices. Withdrawing consent must be as easy as giving
it - GDPR Article 7(3), and the same duty appears in Thailand's PDPA, Indonesia's PDP Law and
India's DPDP Act. Under CCPA it is also the visitor's standing route to opting out of the
sale or sharing of their data. The floating cookie icon is one way in; give them another,
because a footer link is where people look for it.

Offer the user a "Cookie settings" link and ask where they want it: the site footer (the
usual place, present on every page), the privacy or cookie policy page, or both. It calls
`katla.open()`, which opens the preference center:

```html
<button type="button" onclick="window.katla?.open?.()">Cookie settings</button>
```

In React:

```tsx
<button type="button" onClick={() => (window as any).katla?.open?.()}>
  Cookie settings
</button>
```

`window.katla` appears a moment after the script loads, so always call it with optional
chaining. Style it like the other footer links - a `<button>` reset to look like a link is
fine, and keeps it keyboard-reachable.

**Never turn the floating icon off without this link.** `showSettingsButton: false` in
`katla_update_site_settings` hides the icon after a visitor has chosen; without a link, they
are left with no way to change their mind, and the tool says so. Add the link first, publish,
then switch the icon off if the user wants it gone.

## Step 5: Match the banner to the site

Do this on every install, without waiting to be asked. Until the site has colours of its
own the banner wears Katla's default violet, and a user who is not offered their own
colours here has to find them on the dashboard's **Appearance** page afterwards. Offering
is the step; saving is the user's decision. Never save colours the user has not seen and
agreed to.

The banner's look is configuration, not code: never restyle it with CSS overrides from the
site. Instead read the site's design and set the banner's colours, corner radius and theme
with `katla_update_site_settings`, so it looks like part of the site.

The colours are the site's **branding**, not the banner's alone: Katla's accessibility
widget wears the same ones, so saving them restyles both. Say so when you show the user the
values.

1. **Check what Katla already has.** Call `katla_get_site` now, even if you called it
   before the scan: it returns `branding` (the saved colours, or null if none have been
   set) and, only once a scan has completed, `brandingSuggestion`: a light and dark palette
   read off the published homepage and already adjusted to the contrast rules in step 6.
   `katla_get_install_snippet` reports the same under `branding`. If `branding` is null and
   there is a suggestion, offer it as the starting point and compare it with what you find
   in the code in step 2 - the code wins where the two disagree, since the suggestion comes
   from the live site and may predate the changes being made now. If there is no suggestion
   (the site has not been scanned yet, or the user dismissed it), work from the code alone.
   If `branding` is already set, the user chose those colours: say the banner already wears
   them, and change them only if they ask.
2. **Find the design tokens**, in this order, and stop at the first that answers:
   - CSS custom properties in the global stylesheet (`index.css`, `globals.css`,
     `app.css`): shadcn/ui and Lovable projects keep `--primary`, `--background`,
     `--foreground`, `--muted-foreground`, `--radius` there, in HSL or OKLCH, with a
     `.dark` block for dark mode.
   - The Tailwind theme (`tailwind.config.*` `theme.extend.colors`/`borderRadius`, or
     `@theme` in the CSS for Tailwind 4).
   - The classes on the site's main call-to-action button, cards and body text.
   - The published site's CSS, if the code is not available.
3. **Map them onto the six brand colours**, for light and, if the site has a dark mode,
   dark:

   | Colour | Take it from | Banner | Accessibility widget |
   |---|---|---|---|
   | `primary` | The main call-to-action or brand colour (`--primary`) | Accept all, cookie icon | The button |
   | `secondary` | The site's text colour or a neutral dark (light) / light (dark) | Reject | - |
   | `accent` | Same as `primary`, unless the site has a distinct accent | "Always on" badge | - |
   | `background` | The card or popover surface (`--card`, `--popover`, or `--background`) | The card | The panel |
   | `foreground` | Body text (`--foreground`) | Text | Panel text |
   | `muted` | Secondary text (`--muted-foreground`) | Descriptions, links | - |

   Convert every value to 6-digit hex (`#rrggbb`); the tool rejects anything else. Resolve
   HSL/OKLCH triplets and `var()` references to their final colour. The accessibility widget
   uses the light colours unless the banner theme is `dark`.
4. **Radius:** the banner's corner is the dial x 2 px. Take the site's card or dialog radius
   in px and halve it (`--radius: 0.5rem` = 8px, so `radius: 4`; square corners, `0`;
   anything above 20px, `10`).
5. **Theme:** `system` if the site switches with the OS or has a `.dark` theme, otherwise
   `light` (or `dark` for a dark-only site). Only fill `colors.dark` when the site has one.
6. **Check contrast:** `foreground` on `background` should reach 4.5:1 and `primary` on
   `background` 3:1. If the brand colour is too pale, take a darker shade from the same
   scale rather than a different hue.
7. **Show the user** the values and where each came from, ask whether to use them, and
   save only once they agree. If they decline, leave the defaults and tell them the colours
   can be set later on the **Appearance** page. Layout, position and blocking are choices,
   not something to infer: keep the defaults unless the user asks.

Without the MCP server, the colours are on the site's **Appearance** page in the Katla
dashboard, which also offers the scan's suggestion; radius, theme and layout are on the
**Cookie banner** page.

With the SDK the banner is the site's own components, so none of this applies there.

## Step 6 (optional): Embed the privacy and cookie policy

Katla generates the policy from the scan, in 13 languages, and keeps it in step with the
cookies it finds. `katla_get_policy` shows what is set up and returns the embed; the text
itself cannot be edited, but four things can be adjusted:

| To change | How |
|---|---|
| Company details (name, contact email, DPO, address, registration number) | `katla_generate_privacy_policy`. The full policy needs a name and contact email first - ask the user for them, never invent them |
| Which policy: full, cookie policy only, or just the cookie table | `format` in the embed URL (`katla_get_policy` or `katla_get_install_snippet` builds it) |
| Language | `locale` in the embed URL; omit it to follow each visitor's browser |
| Where the banner links to it | `privacyPolicyUrl` / `cookiePolicyUrl` with `katla_update_site_settings` |

Put the embed on the site's privacy or cookie policy page (create the page and route if it
does not exist, linked from the footer next to the "Cookie settings" link):

```html
<div id="katla-policy"></div>
<script src="https://cdn.katla.app/{siteId}/policy.js"></script>
```

In a React SPA, render the `<div id="katla-policy">` on the policy route and load the script
once that route has mounted - or fetch the `.json` URL and render its `policy.markdown` with
the site's own markdown renderer. Then set the page's URL as `privacyPolicyUrl` (and
`cookiePolicyUrl`) so the banner links to it.

**Style it like the rest of the site.** The embed writes plain, unstyled HTML (`h1`-`h3`,
`p`, `ul`, `li`, `table`, `strong`, `em`) straight into the page - no shadow DOM, no
stylesheet of its own - so it inherits whatever the site's CSS does to those elements. Look
at how the site styles its other long-form pages (terms, about, blog posts) and match them:

- **Tailwind with `@tailwindcss/typography`:** give the wrapper the site's prose classes,
  e.g. `classes: { wrapper: 'prose prose-neutral dark:prose-invert max-w-none' }`.
- **Tailwind without it:** Preflight strips heading sizes, list bullets and table borders,
  so the policy will look like one block of text. Map each element to the classes the site
  uses for the same thing elsewhere (headings, body copy, links, tables).
- **Plain CSS or a CSS framework:** add rules scoped to `#katla-policy` in the site's
  stylesheet, using its existing font, colour and spacing variables.
- **Always** give the table visible structure (cell padding, row borders, left-aligned
  header) and let it scroll horizontally on small screens - the cookie table is wide.

Classes and inline styles are set on `window.KatlaPolicy` in a script placed *before* the
policy script (keys: `wrapper, h1, h2, h3, p, ul, li, table, thead, tbody, tr, th, td,
strong, em`). See `references/policy-embed.md`.

## Step 7: Publish, then scan again

The tag does nothing until the site is published with it, and Katla only knows the install
is live once a scan has seen the guard on the published pages. Any scan from before the
install could not see it. So end every install by asking the user for one more scan:

1. **Ask the user to publish** or deploy the site with the changes made here.
2. **Tell them what to write** in this chat once it is live, for example:

   > I have published. Scan {domain} with Katla again and check that the guard is working.

   Mention that they can also start the scan from the Katla dashboard, and that the banner
   and policy pick up what it finds on their own, with no code change. Do not start the
   scan yourself before they say the site is live: a scan of the old version finds no tag
   and still counts against their plan.
3. **When they ask, scan and read the result.** `katla_scan_site`, then poll
   `katla_get_scan_status` until the newest scan is `completed`. Its `guard` is what the
   scan saw of the tag on the published pages:
   - `status: "installed"` and `siteIdMatches: true` - the guard is live. Tell the user,
     with `pagesWithGuard` out of `pagesChecked`.
   - `status: "missing"` - the tag is not on the published homepage. Usually the site was
     not republished. Otherwise check the deployed HTML: the tag must be in `<head>`, and
     not blocked by a Content-Security-Policy (see Common mistakes).
   - `siteIdMatches: false` - the page carries another Katla site's tag. Replace it with
     the one from `katla_get_install_snippet`.
   - `status: "not_checked"` - the scan did not look, for the reason in `summary`. Do not
     report the install as confirmed.
4. **Tell them which cookies it found**, with `katla_list_cookies`. Tags added in this
   session show up here for the first time.
5. **Check it as a visitor.** `katla_check_cookies` on the live URL shows whether anything
   non-essential is still set before a visitor chooses. It is free and uses no scan.

If the plan has no scans left (`scansRemaining` is 0 in `katla_get_scan_status`), say so:
`katla_check_cookies` still works, and the scan can wait for the next period.

## Verify the install

Step 7's scan confirms the tag is on the published pages. To see it as a visitor does:

1. Open the published site in a private window: the banner should appear.
2. Before choosing, check DevTools → Application → Cookies: no analytics or marketing
   cookies should be set.
3. Accept, reload: the banner stays closed and a `_katla_consent` cookie exists.
4. With the MCP server, `katla_check_cookies` on the live URL runs a one-page audit, and
   `katla_check_consent_mode` checks the Google Consent Mode setup.

## Common mistakes

- Placing the tag at the end of `<body>` or loading it from JavaScript: cookies set before it
  runs are not blocked.
- Installing the widget and the SDK's `KatlaProvider` on the same page.
- A `Content-Security-Policy` that blocks it. Allow `https://cdn.katla.app` in `script-src`
  and `https://consent.katla.app` in `connect-src` (and `https://cdn.katla.app` in `img-src`
  on the free plan).
- Using a script optimizer that defers or combines the Katla tag.
- Scanning a preview or localhost URL instead of the published domain.
