---
title: "Styling the export"
description: "The HTML the website export writes for each kind of content, and the CSS selectors that reach it."
---

The exported website is plain HTML and one stylesheet,
`templates/style.css`, which replaces the built-in one. This page describes
the HTML the export writes for each kind of content, so you know which
selector reaches the thing you want to change.

## Who writes the markup

Two things write the HTML of a page, and only one of them is guaranteed.

| Written by | What | Can you rely on it? |
| --- | --- | --- |
| **Provenance itself** | The `<body>` attributes (with the built-in layout), embeds, query results, heading numbers, list-item ids, captions, links and references, and the markers for missing values. | **Yes.** It is the same whatever templates the project has. Everything marked *always* on this page. |
| **A template** | Everything else: the page around the content (`_layout.tmpl`), and how each entity is laid out (`<Type>.tmpl`). | Only for the built-in templates, described below. A project template writes whatever markup it likes, so your stylesheet styles what *you* wrote. |

So when a type has a project template, look at that template, not at the
built-in markup below, for the entity's own elements.

## Which type, which page

Every place an entity can appear says which type it is, so a rule can reach
one type and leave the others alone.

| Where | Marker | Written by |
| --- | --- | --- |
| The whole page | `<body data-page="entity" data-type="Requirement" data-entity="REQ-0001">` | The built-in layout. A project `_layout.tmpl` must write it itself (below). |
| An embedded entity, `![[ID]]` or a query block's `render: full` | `<section class="embed" data-entity="REQ-0001" data-type="Requirement">` | Provenance, **always**. |
| An entity drawn by the built-in template | `<article class="entity" id="REQ-0001" data-type="Requirement">` | The built-in entity template. |
| An entity drawn by a project template | Whatever the template writes. Write `data-type="{{.Type}}"` on its outer element to match the built-in. | You. |

`data-page` is one of:

| `data-page` | The page | `data-type` | `data-entity` |
| --- | --- | --- | --- |
| `entity` | An entity's own page, `entities/<ID>.html` | The entity's type | Its ID |
| `index` | The main page, `index.html` | The document's type, when the export is [scoped]({{< relref "docs/concepts/exporting#scope" >}}) to one; absent otherwise | Its ID, likewise |
| `type` | A type's page, `entities/<Type>.html` ([`pages: false`]({{< relref "docs/concepts/schema#entities-without-pages" >}})) | That type | absent |

A project layout writes the same attributes from `.Page`, `.Type` and
`.Entity` ([Layout]({{< relref "docs/concepts/templates#layout" >}})):

```go-html-template
<body data-page="{{.Page}}"{{with .Type}} data-type="{{.}}"{{end}}{{with .Entity}} data-entity="{{.}}"{{end}}>
```

## Recipes

### Remove section numbers from requirements only

A requirement can appear on its own page and embedded in a document. To hide
its section numbers in both places, and nowhere else:

```css
/* A requirement's own page. */
body[data-type="Requirement"] main .section-number,
/* A requirement embedded anywhere: a document, a query block. */
.embed[data-type="Requirement"] .section-number {
  display: none;
}
```

The numbers are still counted: a document that embeds a requirement still
numbers the sections after it as if the requirement's headings had numbers,
and a [reference]({{< relref "docs/concepts/documents#section-and-item-references" >}})
to one of its headings still shows its number. Hiding is display only.

Each selector matches everything *inside* the requirement too. If a
requirement embeds another entity whose numbers you want to keep, put them
back:

```css
.embed[data-type="Requirement"] .embed:not([data-type="Requirement"]) .section-number,
body[data-type="Requirement"] main .embed:not([data-type="Requirement"]) .section-number {
  display: inline;
}
```

### Other things you might want

| To | CSS |
| --- | --- |
| Hide every section number | `.section-number { display: none; }` |
| Show section numbers in documents only | `.section-number { display: none; }` then `body[data-type="Document"] main .section-number, .embed[data-type="Document"] .section-number { display: inline; }` |
| Hide numbers on a requirement's own page, but keep them when a document embeds it | `body[data-type="Requirement"] main .section-number { display: none; }` |
| Box every embedded requirement | `.embed[data-type="Requirement"] { border: 1px solid #ccc; padding: 8px 16px; }` |
| Hide the field table of embedded entities (built-in template) | `.embed .fields { display: none; }` |
| Hide the type label above each entity (built-in template) | `.entity-type { display: none; }` |
| Style one entity | `[data-entity="REQ-0001"]`: its page's `<body>` and every embed of it |
| Show the number after "Figure" in bold | `.caption-number { font-weight: 600; }` |
| Letters for nested steps, then roman numerals | `ol ol { list-style-type: lower-alpha; } ol ol ol { list-style-type: lower-roman; }` (built in) |
| Style references to sections differently from links to entities | `a.section-ref { … }` and `a.ref:not(.section-ref):not(.item-ref):not(.xref) { … }` |

## The page

The built-in layout, `_layout.tmpl` if the project has none:

```html
<body data-page="entity" data-type="Requirement" data-entity="REQ-0001">
<header class="site-header">
  <a class="site-name" href="../index.html">NeuroPulse</a>
  <span class="site-code">NEURO</span>
</header>
<main>
  … the page's content …
</main>
<details class="revisions">           <!-- only in a git repository -->
  <summary>Revision history</summary>
  <table>…</table>                     <!-- tag names in <span class="tag"> -->
</details>
<footer class="site-footer">Generated by Provenance from commit <code>…</code> …</footer>
</body>
```

## An entity, with the built-in template

A type with no `templates/<Type>.tmpl` is drawn like this, on its own page
and wherever it is embedded:

```html
<article class="entity" id="REQ-0001" data-type="Requirement">
  <p class="entity-type">Requirement</p>
  <h1><span class="id">REQ-0001</span> Amplitude limit</h1>
  <table class="fields">
    <tbody>
      <tr><th>status</th><td>approved</td></tr>          <!-- one row per field -->
      <tr class="incoming"><th>Verified by</th><td>…</td></tr>  <!-- one per incoming link -->
    </tbody>
  </table>
  <p class="table-row">Row 11 of <code>data/complaints.csv</code>.</p>  <!-- CSV rows only -->
  <h2>statement</h2>                                     <!-- a body field, by its label -->
  <div class="body">… the Markdown, as HTML …</div>
</article>
```

A field's label is its `friendly_name`, or its name as written.

### Field values

| Value | HTML |
| --- | --- |
| Not set | `<span class="absent">—</span>` |
| Not valid for its field | `<span class="invalid-value">heavy</span> <span class="invalid">not a number</span>` |
| String, date, number, boolean, enum | The text, unwrapped |
| Text (Markdown) | `<div class="text">…</div>` |
| Link | `<a class="ref" href="REQ-0002.html" title="…">REQ-0002</a>`, comma-separated |
| List of values | `<ul class="items"><li>…</li></ul>`, or `<span class="absent">no items</span>` |
| List of rows | `<table class="list">`, a column per sub-field, or `<span class="absent">no rows</span>` |

## Embeds and query blocks

**Always.** Provenance wraps every entity it shows in full in a section that
names it and its type. The entity inside is drawn by its type's template,
with its headings moved down to the depth it is shown at (its `<h1>` becomes
an `<h3>` under a document's `<h2>`):

```html
<section class="embed" data-entity="REQ-0001" data-type="Requirement">
  … the entity, as its template draws it …
</section>
```

| Content | HTML |
| --- | --- |
| `![[REQ-0001]]` | The `<section class="embed" …>` above |
| `![[REQ-0404]]`, no such entity | `<p class="embed-missing"><span class="id unresolved-id">REQ-0404</span> <span class="unresolved">unresolved</span></p>` |
| A query block, `render: full` (the default) | `<div class="query">` holding one `<section class="embed" …>` per result |
| A query block, `render: id` or `render: field:…` | `<ul class="query"><li><a class="ref" …>…</a></li>…</ul>` |
| A query block with no results | `<p class="query-empty">No Requirement matches this query.</p>` |
| A `mermaid` or `drawio` block this version can't draw | `<div class="unrendered"><p class="unrendered-note">…</p><pre><code class="language-mermaid">…</code></pre></div>` |
| Any other fenced block | `<pre><code class="language-yaml">…</code></pre>` |

## Headings and section numbers

**Always**, on a page about an entity (`data-page="entity"`, or the main page
of a scoped export). Every heading below the page's title, `<h2>` to `<h6>`,
starts with its number, counted after embeds are placed:

```html
<h2 id="preparation"><span class="section-number">1</span> Preparation</h2>
<h3><span class="section-number">1.1</span> Equipment</h3>
```

- The number is in `<span class="section-number">`, followed by a space.
  Hide or restyle it with `.section-number`.
- A heading tagged `{#preparation}` gets the tag as its `id`: `preparation`
  on its entity's own page, `<ID>-preparation` (such as
  `VER-0001-preparation`) where the entity is embedded.
- The page's title, its `<h1>`, has no number. The main page and type pages
  number no headings: they list entities, not sections.
- Headings a template writes are numbered too, such as the built-in
  template's field labels.

## Numbered lists

Ordered lists are the browser's `<ol>`; an item tagged `{#zero-probe}` gets
the tag as its `id`, like a heading:

```html
<ol>
  <li>Fill the tank.</li>
  <li>Warm up the equipment.
    <ol>
      <li>Power on the simulator.</li>
      <li id="zero-probe">Zero the probe.</li>   <!-- referred to as item 2(b) -->
    </ol>
  </li>
</ol>
```

The number a reference shows (`2(b)`, `2(b)(iii)`) is counted by
Provenance; the number beside the item is drawn by the browser from
`list-style-type`. The built-in stylesheet draws nested lists as letters,
then roman numerals, to match. Keep those two rules if you write your own
stylesheet, or the page and its references disagree.

## Captions

**Always.**

```html
<figure class="captioned captioned-figure" id="caption-control-loop">
  <p><img src="../assets/control-loop.svg" alt="Control loop"></p>
  <figcaption><span class="caption-number">Figure 1</span> The control loop.</figcaption>
</figure>
```

The kind is in the class: `captioned-figure`, `captioned-table` (its
caption comes first, above the table) or `captioned-equation`. The `id` is
`caption-` and the caption's `id`; a caption without one has none.

## Links and references

**Always.**

| Markdown | HTML |
| --- | --- |
| `[[REQ-0001]]`, or `link` in a template | `<a class="ref" href="REQ-0001.html" title="Amplitude limit">REQ-0001</a>` |
| `[[REQ-0001#status]]` | `<a class="ref" …>approved</a>` |
| `[[#fig]]`, `[[ID#fig]]`: a caption | `<a class="ref xref" href="#caption-fig">Figure 1</a>` |
| `[[#setup]]`: a tagged heading | `<a class="ref section-ref" href="#setup">Section 1</a>` |
| `[[#step]]`: a tagged list item | `<a class="ref item-ref" href="#step">item 2(b)</a>` |
| `[[VER-0001#setup]]`, not on this page | `<a class="ref section-ref" href="VER-0001.html#setup">Section 1 of VER-0001</a>` |
| A link to an entity outside the export's [scope]({{< relref "docs/concepts/exporting#scope" >}}) | The same text in a `<span>` with the same classes and `out-of-scope` added, not a link |
| A link to nothing | `<span class="id unresolved-id">REQ-0404</span> <span class="unresolved">unresolved</span>` |

A project `_cite.tmpl` replaces how `[[ID]]` and `[[ID|text]]` are drawn
([Citation style]({{< relref "docs/concepts/documents#citation-style" >}})).

## The main page and type pages

The built-in main page:

```html
<h1>NeuroPulse (NEURO)</h1>
<h2>Requirement</h2>                       <!-- one per type; a link for a type page -->
<p class="table-source">Read from <code>data/complaints.csv</code>.</p>  <!-- CSV types -->
<table class="entity-list">…</table>
```

The built-in page of a type with `pages: false`:

```html
<article class="type-page" id="SeverityLevel">
  <p class="entity-type">Type</p>
  <h1>SeverityLevel</h1>
  <table class="list type-entities">        <!-- entities without bodies: a row each -->
    <tr id="SEV-0003"><td class="id">SEV-0003</td>…</tr>
  </table>
  <!-- or, entities with bodies: each drawn by its template -->
  <section class="type-entity">…</section>
</article>
```
