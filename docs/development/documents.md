# Authoring documents

Pressroom lays out one complete HTML document with Chromium's print engine. This page explains
how to write that document: its resources, fonts, page breaks, headers and footers, and scripts.
The [request options](api.md#post-v1render) set the paper.

## Documents are self-contained

A request carries the whole document as one string, with its resources inside it:

- **Styles** go in `<style>` elements or `style` attributes.
- **Images, fonts and other resources** must be `data:` URLs. Pressroom blocks network
  (`http:`, `https:`), local-file and `blob:` resources, also inside nested frames and `srcdoc`
  documents.
- **A `<base>` element** in the document, or in a header or footer template, gets the request
  rejected with 400.
- **Forms, popups, downloads, workers, service workers and storage** are disabled. Scripts run
  only when the caller's API key allows them: see
  [JavaScript and readiness](#javascript-and-readiness).

When an image is missing from the PDF, check that its source is a `data:` URL. The server logs
how many resources it blocked in each render.

Inline SVG suits logos, marks and charts, and keeps a document small. The
[sample document](../../samples/orders-bilingual.html) uses an inline SVG mark.

## Fonts

A release packages **Noto Sans** in regular, bold, italic and bold italic, and Pressroom adds
these faces to every document. Select them in CSS:

```css
:root { font-family: 'Noto Sans', sans-serif; }
```

Embed any other font in the document, as an `@font-face` rule with a `data:` source. A font
installed on the server can also resolve, but then the output depends on that machine. `GET
/v1/info` lists the packaged faces.

Pressroom waits until the document's fonts are ready before it prints, within the render's
deadline.

## Page size, margins, and breaks

Set the paper through the request options:

- `options.pageSize` selects `Letter` (the default), `Legal`, `Tabloid`, `A3`, `A4`, `A5`, or
  custom dimensions such as `{ "width": "8.5in", "height": "11in" }`. `options.landscape`
  turns the page.
- `options.margins` default to 20 mm at the top and bottom and 15 mm at the left and right. A
  length is in `px`, `mm`, `cm` or `in`, and the margins must leave a printable area.
- `options.preferCssPageSize` set to `true` lets the document's own `@page { size: … }` rule win
  over `pageSize`.
- `options.scale` shrinks or enlarges the rendering, from 0.1 to 2.

Control page breaks with the CSS fragmentation properties:

```css
h2 { break-after: avoid; }         /* keep a heading with what follows */
tr, figure { break-inside: avoid; } /* keep a row or figure on one page */
.appendix { break-before: page; }   /* start a section on a new page */
```

Chromium repeats a table's `<thead>` on each page the table spans.

## Print and screen media

A render uses print media by default, so `@media print` rules apply. The sample document has a
tinted background on screen, for previewing in a browser, and a white one in print. Set
`options.media` to `"screen"` to print a document as a browser displays it. Background colors and
images print by default. Set `options.printBackground` to `false` to leave them out.

## Headers and footers

Running headers and footers are separate HTML templates in the request. By default the header is
empty and the footer prints "Page N / M" at the bottom right, in 10px text with 20px of padding
on the right:

```json
"options": {
  "displayHeaderFooter": true,
  "headerTemplate": "<div></div>",
  "footerTemplate": "<div style=\"font-size: 10px; text-align: right; width: 100%; padding-right: 20px;\">Page <span class=\"pageNumber\"></span> / <span class=\"totalPages\"></span></div>"
}
```

Chromium fills elements that have the classes `date`, `title`, `url`, `pageNumber` and
`totalPages`. A template follows its own rules:

- **Styles.** Give each template its own inline styles, with an explicit `font-size`. The page's
  CSS does not reach it. Leave room for it in the top and bottom margins.
- **Scripts and resources.** Scripts in a template do not run, and resources other than `data:`
  are blocked. A template that tries to load an external resource can fail the render.
- **Fonts.** A template uses the server's system fonts. The packaged Noto Sans faces reach the
  document body only, and an `@font-face` rule inside a template has no effect in the packaged
  Chromium. Name a generic family such as `sans-serif`, and check the result on the server.

The default header, an empty `<div></div>`, replaces Chromium's built-in header and leaves the
footer on. Supply `options.headerTemplate` for a header of your own. Set
`options.displayHeaderFooter` to `false` for a document that draws its own running elements.

## Accessibility and navigation

Pressroom asks for a **tagged PDF** by default, so the headings, lists, tables and alternative
text of well-structured HTML become structure in the PDF. It also builds a **document outline**
(bookmarks) from the headings. `options.tagged` and `options.outline` turn these off.

For accessible output, set `lang` on `<html>`, give the document a `<title>`, use heading levels
in order, add `alt` text, and mark table header cells with `<th scope>`. Tags alone do not make a
PDF conform to PDF/UA: validate the result when you need conformance.

## Locale and time zone

`options.locale` (default `en-CA`) and `options.timeZone` (default `America/Edmonton`) set the
browser's locale and time zone for the render. They change `Intl` formatting and `Date` in
documents that run scripts, and the `date` value in header and footer templates. The locale must
be a culture name that .NET knows, and the time zone an IANA identifier.

## JavaScript and readiness

Scripts run only when the calling application's API key has `AllowJavaScript: true`: see
[Configuration](../configuration.md#javascript-permission). The document can then run inline
scripts and scripts from `data:` URLs, which is how chart libraries and client-side templates
work. A request turns scripts off for one render with `"options": { "javaScript": false }`. It
cannot turn them on for a key that lacks the permission.

Pressroom prints once the document has loaded and its fonts are ready. A document that draws
later can tell Pressroom when it is ready:

```json
"options": {
  "readiness": {
    "selector": "#chart svg",
    "function": "window.readyToPrint === true",
    "delayMs": 0,
    "timeoutMs": 5000
  }
}
```

- `selector` waits until a matching element is visible.
- `function` waits until a JavaScript expression is truthy. It needs scripts to be on, or the
  request gets 400.
- `delayMs` adds a fixed pause after the other conditions.
- `timeoutMs` is the time for all three together. When it runs out, the render fails with 422.

The readiness wait is part of the request's processing deadline. A script error in the page also
fails the render with 422.

## The sample document

[`samples/orders-bilingual.html`](../../samples/orders-bilingual.html) shows these techniques in
one short template: Noto Sans in all four styles, a `@page` rule, inline SVG, a table whose rows
have `break-inside: avoid`, a section with `break-before: page`, print and screen media rules, and
English and French text. Its companion
[`orders-bilingual.request.json`](../../samples/orders-bilingual.request.json) is ready to send.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| An image or icon is missing | It is referenced by URL and not embedded as `data:`. |
| Text appears in a system font | The font is not embedded as an `@font-face` rule with a `data:` source, or the text is in a header or footer template. |
| A chart area is empty | Scripts are off for the key, or the page printed before the chart was drawn. Give the key `AllowJavaScript` and add a readiness condition. |
| 400 for `options.readiness.function` | Scripts are off for this request. |
| 422 | A readiness condition ran out of time, the page raised a script error, or a template tried to load an external resource. |
| A row or figure is split across pages | Add `break-inside: avoid`. |
| A header overlaps the content | Increase the top or bottom margin. |
