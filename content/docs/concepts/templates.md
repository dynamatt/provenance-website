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
| `templates/<Type>.tmpl` | The built-in entity page | Every entity of that type: its own page, and wherever it is embedded with `![[ID]]` or shown in full by a query block. `Requirement.tmpl` for `Requirement`. Types extending it without a template of their own use it too; see [Types that extend another](#types-that-extend-another). |
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
| `query` | A list of the entities the query selects, in its order, like a many-link. Empty when the query can't run. |
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
at, keeping its `id` and `class`, and numbers it
([Styling the export]({{< relref "docs/concepts/styling#headings-and-section-numbers" >}})). On the entity's own page the `<h1>` stays
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

### Types that extend another

A type that [extends]({{< relref "docs/concepts/schema#extending-a-type" >}})
another and has no template of its own uses the nearest template up its
chain: a verification summary report, which extends Document, is laid out by
`Document.tmpl`.

To change only part of that layout, the parent's template declares a
*block*, a named place with nothing in it, and the subtype's template fills
it. Blocks are Go's own `{{block}}` and `{{define}}`:

```go-html-template
{{/* templates/Document.tmpl */}}
<article class="document" id="{{.ID}}">
<h1><span class="id">{{.ID}}</span> {{title .}}</h1>
{{- block "before-body" .}}{{end}}
<div class="body">{{markdown .Body}}</div>
{{- block "after-body" .}}{{end}}
…
</article>
```

```go-html-template
{{/* templates/VerificationSummaryReport.tmpl */}}
{{define "after-body"}}
<section class="trace-matrix">
<h2>Traceability matrix</h2>
<table>
{{- range .scope}}
<tr><td>{{link .}}</td><td>{{range .verified_by}}{{link .}} {{end}}</td></tr>
{{- end}}
</table>
</section>
{{end}}
```

A template holding nothing but `{{define}}`s, white space and comments is
laid over the template chosen for the type above it, rather than replacing
it. The report keeps Document's heading, status, body and reference list,
and adds its matrix after the body. A type further down the chain can fill
another block over both. A template with anything else in it replaces its
parent's entirely.

The built-in entity page declares two blocks, `before-body` and
`after-body`, around the body, so a type can add to the built-in page
without writing a whole template. Inside either block, `.` is the entity, as
in any type template. The
[Reusable documents]({{< relref "docs/cookbook/reusable-documents" >}})
recipe puts this together.

## Functions

| Function | Result |
| --- | --- |
| `markdown .Field` | The Markdown rendered as HTML, with its wikilinks, query blocks, embeds and captions. Nothing for an empty field. |
| `link .Entity` | A link to the entity's page showing its ID. Marked *unresolved* for a missing entity, and plain text for one outside the export's [scope]({{< relref "docs/concepts/exporting#scope" >}}). Nothing for an empty link. |
| `link .Entity "text"` | The same link showing the text. Any value works, such as a number: `{{link . .TypeCitationIndex}}`. |
| `href .Entity` | The relative URL of the entity's page, for a link of your own. Empty for an entity outside the scope, which has no page. |
| `title .Entity` | The entity's title: the value of the field its type marks `title: true`. Nothing for a type without one. |
| `collect .List "field" …` | Follows each named field (a link, a reverse link or a query field) from every entity in the list, in turn, and gives the entities reached, each once, ordered by ID. `{{range collect .scope "verified_by" "equipment_needed"}}` lists each piece of equipment once, however many protocols need it. The list may be one entity. A name the entities don't have stops the export. |
| `queryof .Entity "field"` | The query of a [query field]({{< relref "docs/concepts/schema#query-fields" >}}), as YAML, for a report to state what it covers. |
| `isa .Entity "Type"` | `true` when the entity's type is that type or [extends]({{< relref "docs/concepts/schema#extending-a-type" >}}) it, so `{{if isa . "Document"}}` holds for a report too, where `{{if eq .Type "Document"}}` doesn't. A type the schema doesn't declare stops the export. |
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
| `.Page` | What kind of page it is: `entity`, `index` or `type`. |
| `.Type` | The type the page is about: the entity's, a type page's, or a scoped main page's document's. Empty on the unscoped main page. |
| `.Entity` | The ID of the entity the page is about. Empty on the unscoped main page and type pages. |

Write `.Page`, `.Type` and `.Entity` on `<body>`, as the built-in layout
does, so the stylesheet can style one type's pages
([Styling the export]({{< relref "docs/concepts/styling#which-type-which-page" >}})):

```go-html-template
<body data-page="{{.Page}}"{{with .Type}} data-type="{{.}}"{{end}}{{with .Entity}} data-entity="{{.}}"{{end}}>
```

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
`style.css` in any exported site. Every class and attribute the export
writes, and recipes such as hiding one type's section numbers, are in
[Styling the export]({{< relref "docs/concepts/styling" >}}).

The stylesheet isn't part of what a page shows for its
*last changed* stamp: restyling the site doesn't change any page's
revision history.

## Errors

Every template is checked when the export starts, including ones no page
uses. A template that can't be parsed, or that fails while rendering,
stops the export with exit code `2` at its file, line and column. When the
failure is inside a nested render, such as a field of an embedded entity,
the error names the template where it started.
