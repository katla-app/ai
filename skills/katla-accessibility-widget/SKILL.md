---
name: katla-accessibility-widget
description: >-
  Installs Katla's accessibility widget (one script tag) on a website: a button that lets
  each visitor set text size, spacing, colour scheme, saturation, cursor and motion for
  themselves, with links to the site's accessibility statement and a way to report a
  barrier. Use when the user wants an accessibility widget, accessibility button or
  accessibility toolbar, mentions "Katla accessibility," "a11y widget," "KatlaA11y,"
  "katla-a11y," or asks for a link to their accessibility statement in a panel. Works on
  any site: Vite, React, Next.js, plain HTML, Lovable, site builders. It is not an audit:
  to find accessibility problems, use Katla's scan results instead.
homepage: https://docs.katla.app/accessibility-widget
---

# Katla Accessibility Widget

One static script adds a launcher button in a corner of the page. It opens a panel where
a visitor can change how the page looks for them: text size, readable font, line and
letter spacing, text alignment, dark/light/high-contrast/inverted colours, saturation,
highlighted links, headings, focus and hover, a large cursor, a reading guide, stopped
animations, hidden images and muted sounds. Choices are kept in that browser's
`localStorage`. The widget is free on every plan.

## What it is not - read this first

The widget does not scan, repair or relabel anything, and it does not make a site conform
to WCAG, EN 301 549 or the European Accessibility Act. Overlays that claim otherwise are
why accessibility professionals distrust the whole category, and a site owner who installs
one believing they are now compliant is worse off than one who knows they are not.

So, whatever you write:

- Never tell the user, or put in the site's copy, that the widget makes the site
  accessible, compliant or conformant. "Lets visitors adjust the page for themselves" is
  the accurate description.
- Never write an accessibility statement that claims conformance because the widget is
  installed.
- If the user wants to know whether the site *is* accessible, that is the scan:
  `katla_get_accessibility` (every plan) and `katla_get_accessibility_findings` (Pro and
  Business) with the Katla MCP server. Offer it alongside the install, and fix real defects
  in the code - that is what actually helps visitors.

## Prerequisites

1. **Choose the site with the user.** The tag carries a Katla site ID, which decides
   whether the panel shows the "Powered by Katla" line (Free and Starter plans; Pro and
   above drop it). With the Katla MCP server, call `katla_list_sites` and confirm the site
   with the user. Never pick one yourself or take the first in the list - not even when the
   account has only one site. If the project's published domain is not in the list, suggest
   adding it with `katla_add_site`. Unlike the consent banner, the widget does not need a
   cookie scan to work.
2. **Ask about the two panel links** (both optional, both recommended):
   - **Accessibility statement** - the site's own page, e.g. `/accessibility`. The European
     Accessibility Act expects one, and a visitor who has just hit a barrier looks for it
     in this panel.
   - **Report a problem** - a contact page or a `mailto:` address.
   Ask the user for them. Never invent an email address. If the site has no statement page,
   offer to create one, but see the rule above about what it may claim.

## Step 1: Get the tag

Call `katla_get_accessibility_widget_snippet` with the site and any of `position`
(`bottom-right` default, or `bottom-left`), `statementUrl` and `feedbackUrl`.
It returns the tag:

```html
<script src="https://cdn.katla.app/a11y.js" data-site="{siteId}" data-statement="/accessibility" data-feedback="mailto:access@example.com" defer></script>
```

Use it exactly as returned. Never guess or invent a site ID.

Without the MCP server, the tag is in the Katla dashboard: open the site's **Accessibility**
page and choose **Install widget**, which also shows a live preview.

**Colours come from the site's branding**, not the tag. The widget fetches them from Katla
on load: the brand's primary colour for the button, its background and text colour for the
panel. They are the same colours as the consent banner, set with `katla_update_site_settings`
`colors` or on the site's **Appearance** page, and a change there reaches the widget without
touching the tag. Both the light and dark brand colours are used: the widget follows the
banner's theme, so with the banner on System the panel matches each visitor's light or
dark setting. With no branding set, the widget uses Katla violet.

So to make the widget match the site, set the branding - follow "Match the banner to the
site" in the katla-widget skill, which starts from the palette Katla read off the homepage
(`brandingSuggestion` in `katla_get_site`). Do not pass `accent` to the snippet tool or add
`data-accent` to the tag unless the user wants this widget to differ from the brand: a
colour on the tag overrides the branding and stops following it.

The widget picks white or dark text for its button to suit the colour, so a pale brand
primary stays readable.

## Step 2: Put it on every page

The widget sets no cookies and blocks nothing, so it does not need to be first in `<head>`
and must not wait for consent. Load it once per page, just before `</body>`:

| Stack | Where |
|---|---|
| Vite, Lovable, plain React SPA | `index.html`, just before `</body>` |
| Plain HTML / multi-page site | Every page, or the shared footer partial |
| Next.js App Router | `app/layout.tsx`: a plain `<script>` as the last child of `<body>` in the root layout |
| Next.js Pages Router | `pages/_document.tsx`, after `<NextScript />` |
| WordPress, Shopify, site builders | The site-wide "custom code before `</body>`" setting |

Vite / Lovable example (`index.html`):

```html
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
    <script src="https://cdn.katla.app/a11y.js" data-site="{siteId}" defer></script>
  </body>
```

Rules that keep the configuration working:

- Keep it a classic `<script>` tag. The widget reads its `data-*` attributes from
  `document.currentScript`, which is `null` for `type="module"` scripts and for code
  bundled into the app, so never import it or add `type="module"`.
- Load it exactly once per page, from the HTML rather than a component that mounts per
  route. Two copies of the tag on one page can mount two launchers.
- Do not gate it behind the consent banner or `KatlaConsent`. It stores only the visitor's
  display preferences, in `localStorage`, and sends nothing about them anywhere.

## Step 3 (optional): Open it from the site's own link

If the site already has an "Accessibility" link in its footer, point it at the panel
instead of leaving two controls that do the same thing. Any of these opens it:

```html
<a href="#katla-a11y">Accessibility</a>
<button type="button" data-katla-a11y-open>Accessibility settings</button>
```

```javascript
window.dispatchEvent(new Event('katla-a11y:open'));
```

The widget binds these once it has mounted; until then an `#katla-a11y` link only changes
the URL hash, which is harmless. Keep a real link to the statement page as well: opening
the panel is not a substitute for publishing the statement.

## Step 4 (optional): Labels, colours and language

The panel is in English by default, and there is no `locale` setting yet. On a site in
another language, translate the labels, so a Swedish shop does not have an English panel in
the corner. Declare `window.KatlaA11y` in a script *above* the tag:

```html
<script>
  window.KatlaA11y = {
    labels: { title: 'Tillgänglighet', open: 'Tillgänglighetsinställningar', reset: 'Återställ allt' },
  };
</script>
<script src="https://cdn.katla.app/a11y.js" data-site="{siteId}" defer></script>
```

Anything set there wins over the tag's attributes. It can also take `theme` and
`darkTheme` (`{ accent, panel, text }`) and `mode` (`system`, `light`, `dark`), which
override the site's branding on this page only - leave
it out unless the user asks for the widget to look different from the banner. Only the labels you pass change; the
rest keep their defaults. Every key, the shortcut and skip-link options, and the colour
rules are in `references/configuration.md`.

## Things that happen on their own

Tell the user about these rather than configuring them:

- **Keyboard access.** A visually hidden "Accessibility settings" button becomes the page's
  first tab stop, and Alt+Shift+A opens the panel from anywhere. Escape closes it and
  returns focus to the launcher.
- **The consent banner.** If Katla's consent widget is in the same corner, the launcher
  stacks above it rather than covering it.
- **The visitor's OS settings.** A visitor with *reduce motion* or *increase contrast*
  switched on starts with stopped animations or high contrast already selected.
- **Hiding it.** A visitor can hide the launcher from the panel; it comes back on reload.

## Verify the install

1. Open the published site: the launcher is in the chosen corner.
2. Reload and press Tab once: "Accessibility settings" appears as the first focus stop.
   Enter opens the panel, Escape closes it, and focus lands on the launcher.
3. Turn on a large text size and a colour scheme, then reload: the choices stick.
4. Check both panel links go to the statement page and the feedback address.
5. With Katla's consent banner installed, check the launcher sits above it, not on it.

## Common mistakes

- Describing the widget as making the site accessible or compliant.
- Bundling the script, loading it as a module, or injecting it from JavaScript that strips
  the `data-*` attributes: it then boots with defaults and no site ID.
- Hard-coding the brand colour as `data-accent` or `theme`: it then stops following the
  site's branding. Set the branding instead.
- A `Content-Security-Policy` that blocks it. Allow `https://cdn.katla.app` in both
  `script-src` and `connect-src` (the widget asks the CDN whether to show branding).
- Putting the tag only on the home page. Visitors hit barriers on every page.
