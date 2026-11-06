---
title: "Templates"
description: "Change how the exported website looks: type templates, named templates, citations, the layout, the main page and the stylesheet."
---

`provenance export website` renders every page from templates. Provenance
has a built-in version of each, so a repository with no templates still
exports a complete site. To change how something looks, add a file to the
repository's `templates/` folder. Each file replaces its built-in
counterpart and nothing else. A project only writes templates for what it
wants to look different.

| File | Replaces | Used for |
| --- | --- | --- |
| `templates/<Type>.tmpl` | The built-in entity page | Every entity of that type: its own page, and wherever it is embedded with `![[ID]]` or shown in full by a query block. `Requirement.tmpl` for `Requirement`. |
| `templates/<name>.tmpl` | Nothing | Only where a query block chooses it. The name is lower-case and hyphenated, such as `requirement-checklist`. See [Custom templates]({{< relref "docs/concepts/documents#custom-templates" >}}). |
| `templates/_cite.tmpl` | The plain ID of `[[ID]]` | Every inline citation. See [Citation style]({{< relref "docs/concepts/documents#citation-style" >}}). |
| `templates/_layout.tmpl` | The built-in layout | The HTML around every page: head, header, footer, revision history. |
| `templates/_index.tmpl` | The built-in main page | The site's `index.html`. |
| `templates/_<Type>.tmpl` | The built-in type page | The page of a type with `pages: false`, `entities/<Type>.html`. See [Type pages](#type-pages). |
| `templates/style.css` | The built-in stylesheet | Every page. |
| `templates/reports/<name>.tmpl` | Nothing | A report of `provenance validate --output <name>`, not part of the site. See [Custom reports]({{< relref "docs/concepts/rules#custom-reports" >}}). |

Other files in `templates/`, such as a README, are ignored, as is a type
template whose name matches no type in the schema.

Templates are Go [`html/template`](https://pkg.go.dev/html/template)
files. They only lay out HTML: converting Markdown and linking to other
entities are template functions, so a template never builds a link or a
path itself.

## Type templates

A type template receives one entity as `.`. It reads a field, an incoming
link or a list row's sub-field by its name in the schema: `verified_by` is
`.verified_by`. The engine's own keys, below, are capitalised, so they never
clash with a field: a type may declare `title`, `resolved` or `file` like
any other field. An entity's title is not a key: it is the value of the
field its type marks [`title: true`]({{< relref "docs/concepts/schema#entity-types" >}}),
which the `title` function gives, `{{title .}}`.

| Accessor | Value |
| --- | --- |
| `.ID`, `.Type` | The entity's ID and type name. |
| `.Body` | The Markdown under the frontmatter, or the field declared `body: true`. |
| `.<field>` | Every field the schema declares, set or not, by its name. Calculated fields hold their computed value. |
| `.<facet>` | Every incoming link, by its `reverse_name`, such as `.implemented_by`: a list of the entities linking here. |
| `.Resolved` | `true` for an entity that exists. See [links](#links). |
| `.LastChangedSHA`, `.Revisions` | The page's [git stamps](#layout), also given to the layout. |
| `.Citations` | What the page cites, on the entity the page is about. See [Reference lists]({{< relref "docs/concepts/documents#reference-lists" >}}). |
| `.File` | The file the entity is written in: `.Path`, and `.Line`, where its ID is written. See [Importing data]({{< relref "docs/concepts/importing-data#in-templates" >}}). |

A field the entity doesn't set is present and empty, so `{{with .rationale}}`
and `{{if .rationale}}` work. A name the schema doesn't declare stops the
export, so a misspelt field never renders as an empty gap:

```text
export: templates/Requirement.tmpl:25:7: executing "templates/Requirement.tmpl" at <.rationle>: map has no entry for key "rationle"
```

To read a field that only some entities have, such as a status in a
template shared by several types, use `index`, which gives nothing for a
missing name: `{{with index . "status"}}…{{end}}`.

### Values

| Field type | In a template |
| --- | --- |
| `string`, `text`, `date`, an enum | A string. Render `text` with `{{markdown .statement}}`. |
| `number` | A number. |
| `boolean` | `true` or `false`. |
| `link`, one target | The linked entity, with all its own accessors, or nothing. |
| `link`, many targets | A list of linked entities. |
| `list` of rows | A list of rows, each with its sub-fields by name. |
| `list` with `of:` a type or enum | A list of plain values. |
| `calculated` | The formula's result, or nothing when it is blank or can't be evaluated. |

### Links

A link field holds the linked entities themselves, so a template follows
links without looking anything up:

```go-html-template
{{range .implements}}
<li>{{link .}}: {{title .}}</li>
{{end}}
```

A link to an ID that no entity has still gives an entity, with `.Resolved`
false and every field of its target type empty. A template written for
resolved links renders it without special cases, and `link` marks it
*unresolved*.

### Headings

Write headings relative to the entity: `<h1>` is its title, `<h2>` its
sections. Provenance shifts every heading to the depth the entity is shown
at, keeping its `id` and `class`. On the entity's own page the `<h1>` stays
an `<h1>`. Embedded under a document's `<h2>` section, it becomes an `<h3>`.

### An example

`templates/Requirement.tmpl` from
[provenance-example](https://github.com/dynamatt/provenance-example),
shortened:

```go-html-template
<article class="requirement" id="{{.ID}}">
<h1><span class="req-id">{{.ID}}</span> {{title .}}</h1>
<p class="requirement-meta">
<span class="status status-{{.status}}">{{.status}}</span>
{{- with .parent_requirement}} · refines {{link .}}{{end}}
</p>
<div class="statement">{{markdown .statement}}</div>
<h2>Verified by</h2>
{{- if .verified_by}}
<ul>
{{- range .verified_by}}
<li>{{link .}}: {{title .}}</li>
{{- end}}
</ul>
{{- else}}
<p class="none">Not yet verified.</p>
{{- end}}
</article>
```

## Functions

| Function | Result |
| --- | --- |
| `markdown .Field` | The Markdown rendered as HTML, with its wikilinks, query blocks, embeds and captions. Nothing for an empty field. |
| `link .Entity` | A link to the entity's page showing its ID. Marked *unresolved* for a missing entity, and plain text for one outside the export's [scope]({{< relref "docs/concepts/exporting#scope" >}}). Nothing for an empty link. |
| `link .Entity "text"` | The same link showing the text. Any value works, such as a number: `{{link . .TypeCitationIndex}}`. |
| `href .Entity` | The relative URL of the entity's page, for a link of your own. Empty for an entity outside the scope, which has no page. |
| `title .Entity` | The entity's title: the value of the field its type marks `title: true`. Nothing for a type without one. |
| `short .SHA` | A commit hash shortened to seven characters. |

The usual `html/template` functions, such as `index`, `len`, `eq` and
`printf`, work too.

## Layout

`templates/_layout.tmpl` is the whole HTML page around each page's content.
It receives:

| Accessor | Value |
| --- | --- |
| `.Title` | The page title: the entity's ID and title, or the component's name on the main page. This is the page's, not a field. |
| `.Root` | The path from the page to the site's root, `""` or `"../"`. Prefix links to site files with it: `{{.Root}}style.css`. |
| `.Component.Name`, `.Component.Code` | From the repository's `.component` file. |
| `.Content` | The page's rendered content. `{{template "content" .}}` does the same. |
| `.GitSHA` | The commit the export was generated from. |
| `.ContentHash` | The content hash, ending in `-dirty` for uncommitted changes. |
| `.LastChangedSHA` | The last commit that changed what this page shows, ending in `-dirty` when some of it is uncommitted. |
| `.Revisions` | Every commit that changed what this page shows, newest first, each with `.SHA`, `.Short`, `.Date`, `.Author`, `.Subject` and `.Tags`. |
| `.Citations` | What the page cites, as a type template receives it. Empty on the main page. |

The git stamps are empty outside a git repository. What counts as "what
this page shows" is described in
[Provenance on every page]({{< relref "docs/concepts/exporting#provenance-on-every-page" >}}).

## Main page

`templates/_index.tmpl` is the content of `index.html`, inside the layout.
It receives `.Component` and `.Types`: every entity grouped by type, types
in name order and entities by ID. Each group has `.Type` and `.Entities`,
and each entity the same accessors as in its type template. A type with
`pages: false` also has `.Page`, the link to its [type page](#type-pages),
and is empty otherwise; a type read from a CSV file has `.File`, with the
file's `.Path`:

```go-html-template
<h1>{{.Component.Name}}</h1>
{{range .Types}}
<h2>{{if .Page}}<a href="{{.Page}}">{{.Type}}</a>{{else}}{{.Type}}{{end}}</h2>
<ul>
{{- range .Entities}}
<li>{{link .}} {{title .}}</li>
{{- end}}
</ul>
{{end}}
```

When the export is [scoped]({{< relref "docs/concepts/exporting#scope" >}})
to a document, the document is the main page instead, rendered through its
type template.

## Type pages

A type whose schema says
[`pages: false`]({{< relref "docs/concepts/schema#entities-without-pages" >}})
has one page, `entities/<Type>.html`, instead of a page per entity.
`templates/_<Type>.tmpl` renders it, inside the layout, and receives:

| Accessor | Value |
| --- | --- |
| `.Type` | The type's name. |
| `.Entities` | Its entities, in ID order, each with the same accessors as in its type template. |
| `.File` | For a type read from a CSV file, the file's `.Path`; nil otherwise. |

Every link to one of the entities points at `<Type>.html#<ID>`, so give
each one an element with its ID as `id`:

```go-html-template
{{/* templates/_Definition.tmpl: a glossary */}}
<h1>Glossary</h1>
<dl>
{{- range .Entities}}
<dt id="{{.ID}}">{{title .}}</dt>
<dd>{{markdown .Body}}</dd>
{{- end}}
</dl>
```

Without a template, the built-in page is a table, one row per entity and a
column per field, when none of the type's entities has a body; otherwise
it shows each entity in turn as its type template renders it, one heading
level down. A `_<Type>.tmpl` for a type whose entities have pages of their
own is never used, and `provenance validate` warns of it.

## Stylesheet

`templates/style.css` replaces the built-in stylesheet entirely, and is
written to the site as `style.css`. Start from a copy of the built-in one,
`style.css` in any exported site. The HTML Provenance generates itself uses
these classes:

| Class | On |
| --- | --- |
| `ref` | A link made by `link` or a wikilink. |
| `out-of-scope` | A reference to an entity outside the export's scope, which is plain text. |
| `unresolved` | A reference to an ID no entity has. |
| `query` | A query block's results: a `<ul>` for `render: id` and `render: field:`, a `<div>` for `render: full`. |
| `query-empty` | A query block's *No … matches this query.* |
| `embed` | The `<section>` holding an embedded entity. |
| `captioned-figure`, `captioned-table`, `captioned-equation` | A captioned figure, table or equation, and its caption. |

The stylesheet isn't part of what a page shows for its
*last changed* stamp: restyling the site doesn't change any page's
revision history.

## Errors

Every template is checked when the export starts, including ones no page
uses. A template that can't be parsed, or that fails while rendering,
stops the export with exit code `2` at its file, line and column. When the
failure is inside a nested render, such as a field of an embedded entity,
the error names the template where it started.
