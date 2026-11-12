# Katla Widget - Policy embed

The Katla policy embed renders a fully localized cookie or privacy policy directly on your website using a lightweight JavaScript snippet.

## Basic embed

Add the following HTML to any page where you want the policy displayed:

```html
<div id="katla-policy"></div>
<script src="https://cdn.katla.app/{siteId}/policy.js"></script>
```

Replace `{siteId}` with your site ID. The script fetches the policy content and renders it inside the `#katla-policy` element.

## Formats

Control which policy content is rendered with the `format` query parameter:

| Parameter | Description |
|-----------|-------------|
| `full` | Full cookie + privacy policy (default) |
| `cookie` | Cookie policy only |
| `table` | Cookie table only |

```html
<script src="https://cdn.katla.app/{siteId}/policy.js?format=cookie"></script>
```

## Locale

Set the language with the `locale` query parameter:

```html
<script src="https://cdn.katla.app/{siteId}/policy.js?locale=de-DE"></script>
```

When no `locale` is specified, the script auto-detects the visitor's browser language and falls back to English.

## Custom styling

You can apply CSS classes and inline styles to every generated HTML element by setting `window.KatlaPolicy` before the script loads.

### Supported element keys

| Key | Element |
|-----|---------|
| `wrapper` | Outer `<div>` wrapping all content |
| `h1` | `<h1>` headings |
| `h2` | `<h2>` headings |
| `h3` | `<h3>` headings |
| `p` | `<p>` paragraphs |
| `tableWrapper` | `<div>` around each table, which scrolls sideways when the table is wider than the page |
| `table` | `<table>` elements |
| `thead` | `<thead>` elements |
| `tbody` | `<tbody>` elements |
| `tr` | `<tr>` rows |
| `th` | `<th>` header cells |
| `td` | `<td>` data cells |
| `ul` | `<ul>` lists |
| `li` | `<li>` list items |
| `strong` | `<strong>` bold text |
| `em` | `<em>` italic text |

All keys are optional. Omitted keys produce elements with no class or style attributes, except the `tableWrapper` box: it always carries `max-width:100%; overflow-x:auto`, so on a phone a wide cookie table scrolls inside its box instead of widening the page. A `tableWrapper` style is added after those two, so it can change them.

### Using CSS classes

Ideal for Tailwind or other utility-first frameworks:

```html
<script>
  window.KatlaPolicy = {
    classes: {
      wrapper: "container mx-auto px-4",
      h1: "text-4xl font-bold mb-6",
      h2: "text-2xl font-semibold mt-8 mb-4",
      h3: "text-lg font-medium mt-6 mb-2",
      p: "text-gray-600 leading-relaxed mb-4",
      table: "w-full border-collapse",
      thead: "bg-gray-100",
      th: "text-left p-2 font-medium",
      tr: "border-b",
      td: "p-2",
      ul: "list-disc pl-6 mb-4",
      li: "mb-2 text-gray-600",
      a: "text-blue-600 underline hover:text-blue-800",
      strong: "font-semibold",
      em: "italic",
    }
  };
</script>
<div id="katla-policy"></div>
<script src="https://cdn.katla.app/{siteId}/policy.js"></script>
```

### Using inline styles

For sites without a CSS framework:

```html
<script>
  window.KatlaPolicy = {
    styles: {
      h1: "font-size: 2rem; color: #111;",
      h2: "font-size: 1.5rem; margin-top: 2rem;",
      p: "color: #444; line-height: 1.6;",
      table: "width: 100%; border-collapse: collapse;",
      th: "text-align: left; padding: 8px; background: #f5f5f5;",
      td: "padding: 8px; border-bottom: 1px solid #eee;",
    }
  };
</script>
<div id="katla-policy"></div>
<script src="https://cdn.katla.app/{siteId}/policy.js"></script>
```

### Combining classes and styles

Both `classes` and `styles` can be used together. When both are set for the same element, both the `class` and `style` attributes are applied:

```html
<script>
  window.KatlaPolicy = {
    classes: { table: "w-full" },
    styles: { table: "border-collapse: collapse;" }
  };
</script>
```

## JavaScript API

After the script loads, `window.KatlaPolicy` exposes the following methods:

| Method | Description |
|--------|-------------|
| `render(selector?)` | Re-render the policy into the given CSS selector (defaults to `#katla-policy`) |
| `getMarkdown()` | Returns a Promise that resolves with the raw markdown string |
| `getLocale()` | Returns the detected or configured locale string |

```js
// Re-render into a different element
window.KatlaPolicy.render("#my-policy");

// Get the raw markdown
window.KatlaPolicy.getMarkdown().then(function (md) {
  console.log(md);
});

// Check the active locale
console.log(window.KatlaPolicy.getLocale());
```

## Direct URLs

Policy content is also available as standalone files:

| Format | URL |
|--------|-----|
| HTML | `https://cdn.katla.app/{siteId}/policy.html` |
| JSON | `https://cdn.katla.app/{siteId}/policy.json` |
| Markdown | `https://cdn.katla.app/{siteId}/policy.md` |
| JavaScript | `https://cdn.katla.app/{siteId}/policy.js` |

All direct URLs accept the same `locale` and `format` query parameters.
