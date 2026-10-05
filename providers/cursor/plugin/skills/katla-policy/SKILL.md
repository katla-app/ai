---
name: katla-policy
description: >-
  Puts Katla's generated privacy and cookie policy on a website: a policy page that embeds
  the policy Katla writes from its cookie scan, styled like the rest of the site and linked
  from the footer and the consent banner. Use when the user wants a privacy policy, cookie
  policy or cookie table on their site, mentions "Katla policy," "policy embed,"
  "policy.js," "KatlaPolicy," "#katla-policy," or asks to keep their cookie policy in sync
  with the cookies their site sets. Works on any site: Vite, React, Next.js, plain HTML,
  Lovable, site builders. To install the consent banner itself, use the katla-widget skill
  (katla-sdk only when the user explicitly asks for the SDK or a custom banner).
homepage: https://docs.katla.app/policy-embed
---

# Katla Policy

Katla writes the site's privacy and cookie policy from its cookie scan and the company
details saved in the dashboard, in 13 languages. The policy is served from Katla's CDN and
rendered into the page by a small script, so it updates itself when a new scan finds
different cookies. Nobody edits the text by hand, and neither should you: the four things
that can be changed are listed under [What can be adjusted](#what-can-be-adjusted).

## Prerequisites

1. **The right Katla site.** The policy describes the cookies Katla found on one site, so it
   must be the site for this project. If the user gave you a site ID and domain, use them.
   Otherwise, with the Katla MCP server, call `katla_list_sites` and confirm the site with
   the user. Never pick a site yourself, and never take the first one in the list.
2. **A completed cookie scan.** The cookie policy and cookie table list what the last scan
   found, so a site that has never been scanned renders an empty list. Scans cost against
   the user's plan: ask before starting one with `katla_scan_site`.
3. **Company details, for the full policy.** The `full` format (privacy policy and cookie
   policy together) renders nothing until a company name and contact email are saved. Check
   with `katla_get_policy` (`companyDetailsSaved`). If they are missing, ask the user for
   them and save them with `katla_generate_privacy_policy`, or have the user fill them in
   on the Policies page of the Katla dashboard. Never invent a company name, address,
   registration number or DPO. The `cookie` and `table` formats work without them.

Without the MCP server, the site ID is in the Katla dashboard under **Policies** or
**Install**. Never guess one.

## Step 1: Choose format and language

| `format` | Renders |
|---|---|
| `full` (default) | Privacy policy and cookie policy |
| `cookie` | Cookie policy only |
| `table` | The cookie table only, for a site that writes its own policy around it |

`locale` sets the language (for example `de-DE`, `sv-SE`). Leave it out to follow each
visitor's browser language, falling back to English. On a site that is already translated,
pass the locale of each language version of the page instead.

Both are query parameters on the embed URL. `katla_get_policy` with `format` and `locale`
returns the finished snippet.

## Step 2: Add the policy page

Use the site's existing privacy or cookie policy page if there is one; replace its
hand-written cookie list rather than leaving two lists that disagree. Otherwise create a
page at the path the site's other legal pages use (usually `/privacy` or `/cookie-policy`)
and link it from the footer.

```html
<div id="katla-policy"></div>
<script src="https://cdn.katla.app/{siteId}/policy.js"></script>
```

With a format and language:

```html
<script src="https://cdn.katla.app/{siteId}/policy.js?format=cookie&locale=de-DE"></script>
```

| Stack | How |
|---|---|
| Plain HTML, site builders | The snippet in the page body, where the policy should appear |
| Vite, Lovable, React SPA | Render `<div id="katla-policy">` on the policy route and load the script once that route has mounted (append a `<script>` in an effect, and remove it on unmount). Or fetch the `.json` URL and render `policy.markdown` with the site's own markdown renderer |
| Next.js | A Server Component that fetches the `.json` URL and renders `policy.markdown` with the site's markdown renderer, revalidated hourly. Or `@katla.app/sdk/next/server` (`getCachedPolicy` and `KatlaPolicy`) if the project already uses the SDK |

The policy is not in `<head>` and does not affect consent, so unlike the banner it can load
late.

## Step 3: Style it like the rest of the site

The embed writes plain, unstyled HTML (`h1` to `h3`, `p`, `ul`, `li`, `table`, `strong`,
`em`) straight into the page, with no shadow DOM and no stylesheet of its own. It
inherits whatever the site's CSS does to those elements. Look at how the site styles its
other long-form pages (terms, about, blog posts) and match them:

- **Tailwind with `@tailwindcss/typography`:** give the wrapper the site's prose classes,
  for example `classes: { wrapper: 'prose prose-neutral dark:prose-invert max-w-none' }`.
- **Tailwind without it:** Preflight strips heading sizes, list bullets and table borders,
  so the policy reads as one block of text. Map each element to the classes the site uses
  for the same thing elsewhere.
- **Plain CSS or a CSS framework:** add rules scoped to `#katla-policy` in the site's
  stylesheet, using its existing font, colour and spacing variables.
- **Always** give the table visible structure (cell padding, row borders, a left-aligned
  header) and let it scroll horizontally on small screens. The cookie table is wide.

Classes and inline styles go on `window.KatlaPolicy`, in a script placed **before** the
policy script:

```html
<script>
  window.KatlaPolicy = {
    classes: {
      wrapper: 'mx-auto max-w-3xl px-4',
      h2: 'mt-8 mb-3 text-2xl font-semibold',
      p: 'mb-4 leading-relaxed',
      table: 'w-full border-collapse text-sm',
      th: 'border-b px-3 py-2 text-left font-semibold',
      td: 'border-b px-3 py-2',
    },
    styles: {
      table: 'display: block; overflow-x: auto;',
    },
  };
</script>
<div id="katla-policy"></div>
<script src="https://cdn.katla.app/{siteId}/policy.js"></script>
```

Keys: `wrapper, h1, h2, h3, p, ul, li, table, thead, tbody, tr, th, td, strong, em`. All
are optional. When a key has both a class and a style, both are applied.

## Step 4: Link the banner to it

The consent banner links to the privacy and cookie policy. Once the page is published, set
its full URL as `privacyPolicyUrl` (and `cookiePolicyUrl`, if the cookie policy has its own
page) with `katla_update_site_settings`, or on the Banner page of the dashboard. Without
this the banner's policy link goes nowhere useful.

If the site has a footer, put the policy link there next to the "Cookie settings" link, if
there is one.

## What can be adjusted

| To change | How |
|---|---|
| Company details (name, contact email, DPO, address, registration number) | `katla_generate_privacy_policy`, or the Policies page in the dashboard. Only the fields passed change |
| Which policy is shown | `format` in the embed URL |
| Language | `locale` in the embed URL |
| Where the banner links | `privacyPolicyUrl` / `cookiePolicyUrl` with `katla_update_site_settings` |

The cookie list changes on its own after each scan. If it looks wrong, the fix is in Katla
(rescan, or reclassify a cookie on the Cookies page), not in the site's code.

## JavaScript API

Once `policy.js` has loaded, `window.KatlaPolicy` also has:

| Method | Does |
|---|---|
| `render(selector?)` | Renders the policy again into a selector (default `#katla-policy`), for example after a client-side route change |
| `getMarkdown()` | A promise of the raw markdown |
| `getLocale()` | The locale in use |

## Direct URLs

The same policy is available without the script. All take the same `format` and `locale`
parameters:

| Format | URL |
|---|---|
| HTML fragment | `https://cdn.katla.app/{siteId}/policy.html` |
| JSON (`policy.markdown` and metadata) | `https://cdn.katla.app/{siteId}/policy.json` |
| Markdown | `https://cdn.katla.app/{siteId}/policy.md` |

Fetch them at request time or on a short revalidation, never once at build time and
committed: the point of the generated policy is that it follows the cookies the site
actually sets.

## Check it

Open the published policy page in a private window and confirm:

- the policy renders, in the expected language, with headings and a readable table,
- it looks like the site's other pages, in light and dark mode if the site has both,
- the banner's policy link opens this page.
