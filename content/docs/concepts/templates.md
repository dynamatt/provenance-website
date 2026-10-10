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
| `templates/style.css` | The built-in stylesheet | Every page. |

Other files in `templates/`, such as a README, are ignored, as is a type
template whose name matches no type in the schema.

Templates are Go [`html/template`](https://pkg.go.dev/html/template)
files. They only lay out HTML: converting Markdown and linking to other
entities are template functions, so a template never builds a link or a
path itself.

## Type templates

A type template receives one entity as `.`. Its fields are named after the
schema's fields in PascalCase: `verified_by` becomes `.VerifiedBy`.

| Accessor | Value |
| --- | --- |
| `.ID`, `.Type`, `.Title` | The entity's ID, type name and title. |
| `.Body` | The Markdown under the frontmatter, or the field declared `body: true`. |
| `.<Field>` | Every field the schema declares, set or not. Calculated fields hold their computed value. |
| `.<Facet>` | Every incoming link, by its `reverse_name`, such as `.ImplementedBy`: a list of the entities linking here. |
| `.Resolved` | `true` for an entity that exists. See [links](#links). |
| `.LastChangedSHA`, `.Revisions` | The page's [git stamps](#layout), also given to the layout. |
| `.Citations` | What the page cites, on the entity the page is about. See [Reference lists]({{< relref "docs/concepts/documents#reference-lists" >}}). |

A field the entity doesn't set is present and empty, so `{{with .Rationale}}`
and `{{if .Rationale}}` work. A name the schema doesn't declare stops the
export, so a misspelt field never renders as an empty gap:

```text
export: templates/Requirement.tmpl:25:7: executing "templates/Requirement.tmpl" at <.Rationle>: map has no entry for key "Rationle"
```

To read a field that only some entities have, such as a status in a
template shared by several types, use `index`, which gives nothing for a
missing name: `{{with index . "Status"}}…{{end}}`.

### Values

| Field type | In a template |
| --- | --- |
| `string`, `text`, `date`, an enum | A string. Render `text` with `{{markdown .Statement}}`. |
| `number` | A number. |
| `boolean` | `true` or `false`. |
| `link`, one target | The linked entity, with all its own accessors, or nothing. |
| `link`, many targets | A list of linked entities. |
| `list` of rows | A list of rows, each with its sub-fields in PascalCase. |
| `list` with `of:` a type or enum | A list of plain values. |
| `calculated` | The formula's result, or nothing when it is blank or can't be evaluated. |

### Links

A link field holds the linked entities themselves, so a template follows
links without looking anything up:

```go-html-template
{{range .Implements}}
<li>{{link .}}: {{.Title}}</li>
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
<h1><span class="req-id">{{.ID}}</span> {{.Title}}</h1>
<p class="requirement-meta">
<span class="status status-{{.Status}}">{{.Status}}</span>
{{- with .ParentRequirement}} · refines {{link .}}{{end}}
</p>
<div class="statement">{{markdown .Statement}}</div>
<h2>Verified by</h2>
{{- if .VerifiedBy}}
<ul>
{{- range .VerifiedBy}}
<li>{{link .}}: {{.Title}}</li>
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
| `short .SHA` | A commit hash shortened to seven characters. |

The usual `html/template` functions, such as `index`, `len`, `eq` and
`printf`, work too.

## Layout

`templates/_layout.tmpl` is the whole HTML page around each page's content.
It receives:

| Accessor | Value |
| --- | --- |
| `.Title` | The page title: the entity's ID and title, or the component's name on the main page. |
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
and each entity the same accessors as in its type template:

```go-html-template
<h1>{{.Component.Name}}</h1>
{{range .Types}}
<h2>{{.Type}}</h2>
<ul>
{{- range .Entities}}
<li>{{link .}} {{.Title}}</li>
{{- end}}
</ul>
{{end}}
```

When the export is [scoped]({{< relref "docs/concepts/exporting#scope" >}})
to a document, the document is the main page instead, rendered through its
type template.

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

Two fields or facets of one type whose accessors would be the same, such
as a field and an incoming link both named `verified_by`, also stop the export, as does a field
whose accessor is one Provenance provides, such as `citations` or `revisions`.
`title` is the exception: a `title` field is the entity's `.Title`.
