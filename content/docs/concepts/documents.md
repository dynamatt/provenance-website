---
title: "Documents and query blocks"
description: "Compose documents from live entities with wikilinks and query blocks."
---

A document, such as a system requirements specification, is an entity whose
Markdown body mixes prose with references to other entities. The document
file stores only the references. Every time the document is exported, they
are resolved against the repository as it is then, so editing a requirement
updates every document that shows it, with nothing to copy or keep in sync.

There are two ways to pull entities into a document: wikilinks, for one
entity you can name, and query blocks, for a set of entities chosen by a
condition.

## Wikilinks

Wikilinks work in any Markdown: an entity's body and its `text` fields.

| Syntax | Renders as |
| --- | --- |
| `[[REQ-0001]]` | A link to the entity, showing its ID. |
| `[[REQ-0001\|the amplitude rule]]` | The same link with your own text. |
| `[[REQ-0002#title]]` | The field's current value, linked to the entity. |
| `![[REQ-0003]]` | The whole entity, rendered through its type's template. It must stand alone in its paragraph. |
| `[[#control-loop]]` | The number of a caption in this document, such as *Figure 1*, linked to it. |

An ID that no entity has is shown marked *unresolved*; the export still
succeeds, but `provenance validate` reports it as an error at its line, as
it does an `[[ID#field]]` naming a field the entity's type doesn't have, a
`[[#id]]` with no caption of that id on the page, and an `![[ID]]` written
inside running text, which is shown as a reference rather than embedded.
Captions and references to them are described in
[Images and captions]({{< relref "docs/concepts/exporting#captions" >}}).
An entity that embeds itself, directly or through other embeds,
stops the export, because the document would never end.

## Reference lists

A document can end with a list of everything it cites, without the author
writing the list. Every `[[ID]]`, `[[ID|text]]` and `[[ID#field]]` on the
page is a citation, including those inside entities it embeds or renders
in full through a query block. Embeds (`![[ID]]`), link fields,
`[[#caption]]` references and the entities a query block lists with
`render: id` or `render: field:…` are not citations.

The list is the template's job, like the revision history, so authors only
cite. A type template receives the page's citations as `.Citations`, in
the order they are first cited, each entity once. Each one carries its
position among all the citations, `.CitationIndex`, and among the cited
entities of its own type, `.TypeCitationIndex`. To list references at the
end of every document, add them to `templates/Document.tmpl`:

```go-html-template
{{markdown .Body}}
{{- with .Citations}}
<h2>References</h2>
<ol>
{{- range .}}
<li>{{link .}}: {{.Title}}</li>
{{- end}}
</ol>
{{- end}}
```

`.Citations` is set only on the entity the page is about. It is empty when
the same document is embedded in another or shown by a query, so a
document inside a document doesn't repeat its list. The page layout,
`templates/_layout.tmpl`, receives the same `.Citations`, for a list on
every page. The built-in templates show no list.

External sources, such as standards and papers, are entities too. Give
them a type of their own, such as `Reference`, and cite them like anything
else, `[[REF-0001]]`. A template tells them apart by type, for example to
list them separately and number them:

```go-html-template
{{range .Citations}}{{if eq .Type "Reference"}}
<li id="ref-{{.TypeCitationIndex}}">{{.Title}}</li>
{{end}}{{end}}
```

A cited entity outside the export's scope is listed as its plain ID, and
one that doesn't exist is marked *unresolved*.

## Citation style

How each inline citation looks is the project's choice too. Without a
template, `[[REF-0001]]` shows the ID `REF-0001`. Add
`templates/_cite.tmpl` and every `[[ID]]` and `[[ID|text]]` on a page is
rendered through it instead. It receives the cited entity as in
`.Citations`, with `.CitationIndex`, `.TypeCitationIndex` and
`.CitationLabel`, the `text` after `|` (empty when there is none). Number
the references and keep everything else as before:

```go-html-template
{{- if eq .Type "Reference" -}}
[{{link . .TypeCitationIndex}}{{with .CitationLabel}}, {{.}}{{end}}]
{{- else if .CitationLabel -}}
{{link . .CitationLabel}}
{{- else -}}
{{link .}}
{{- end}}
```

With this template, `[[REF-0001]]` shows `[1]` and `[[REF-0001|clause 7]]`
shows `[1, clause 7]`, numbered in the same order as the list, while
`[[REQ-0001]]` still shows `REQ-0001`. The numbers belong to the page: a
requirement cited inside an embed is numbered in the page embedding it,
and on its own page by its own citations. The file's final line break
isn't part of the citation. `[[ID#field]]` and `[[#caption]]` keep their
usual rendering. On the site's main page, which isn't about one entity,
citations always render as they would without the template.

`.CitationLabel`, `.CitationIndex` and `.TypeCitationIndex` are reserved:
a schema can't declare fields named `citation_label`, `citation_index` or
`type_citation_index`.

## Query blocks

A query block is a fenced code block with the info string `query`. It
selects entities of one type, optionally filters and sorts them, and renders
each one in place.

````markdown
## Requirements

All approved requirements, in author-assigned order:

```query
from: Requirement
where:
  field: status
  operator: equals
  value: approved
order_by: order
```
````

| Key | Meaning |
| --- | --- |
| `from` | The entity type to select, or a list of types such as `[Risk, Requirement]`. Required. |
| `where` | A condition the entities must meet. Without it, every entity of the type is selected. |
| `order_by` | A field, or a list of fields, to sort by. |
| `render` | How each entity is shown: `full`, `id` or `field:<name>`. Default `full`. |
| `template` | The template to use for each result. Use `template` for a single template or `templates` for a per-type map. If omitted, the default template for the entity's type is used. |
| `templates` | A mapping from entity type name to template name, so a query can mix entities of different types and still choose the right template for each result. |

### Conditions

> **More in the next release.** Conditions are also how `provenance
> validate`'s rules are written. The next release adds links followed to
> any depth.

A condition names a `field`, an `operator` and, for most operators, a
`value` or a `value_of`. Operators are words, not symbols:

| Operator | Holds when the field… |
| --- | --- |
| `equals` | has the value. |
| `not_equals` | does not have the value. This includes a field with no value at all. |
| `greater_than`, `less_than` | is above, or below, the value. |
| `greater_or_equal`, `less_or_equal` | is at least, or at most, the value. |
| `not_in` | has none of a list of values, written `value: [a, b]`. This includes a field with no value at all. |
| `exists` | has at least one value. Takes no `value`. |
| `missing` | has no value. Takes no `value`. |
| `duplicated` | has a value another selected entity also has. Takes no `value`. See [Duplicates](#duplicates). |

`value` is a literal. `value_of` names another field to compare with:

```yaml
where:
  field: approved_on
  operator: greater_or_equal
  value_of: submitted_on
```

A list of conditions means all of them must hold. `any_of` means at least
one must hold, and can contain further `any_of`s:

```yaml
where:
  - field: status
    operator: equals
    value: approved
  - any_of:
      - field: order
        operator: less_than
        value: 10
      - field: verified_by
        operator: exists
```

### What a condition can read

| `field:` | Reads |
| --- | --- |
| `status` | A field of the entity. |
| `verified_by` | A reverse link: the entities that link to this one through that `reverse_name`. |
| `id`, `type` | The entity's ID and type. |
| `^execution_date` | Inside a sub-query (below), a field of the entity around it. `^^` reads two levels out. |

`from: any` selects every entity type.

### Rows and linked entities

A `where` on a list field tests each of its rows on its own, and on a link
or reverse link, each entity it reaches. The condition holds when at least
one row or entity matches. Inside, field names are the row's sub-fields or
the linked entity's fields, and `^` reads the entity the condition started
from. This finds evidence recorded with equipment whose calibration was
due before the test:

```yaml
from: VerificationEvidence
where:
  field: equipment_used
  where:
    field: calibration_due_date
    operator: less_than
    value_of: ^execution_date
```

The conditions inside one `where` are about one row or entity. The example
below finds equipment that is both serial `A` and overdue, not one row of
each:

```yaml
where:
  field: equipment_used
  where:
    - field: serial_number
      operator: equals
      value: A
    - field: calibration_due_date
      operator: less_than
      value_of: ^execution_date
```

Two separate `where`s on the same list may each match a different row. A
`where` can contain another, to follow a link from a linked entity:

```yaml
from: VerificationEvidence
where:
  field: tests
  where:
    field: verifies
    where:
      field: status
      operator: equals
      value: deprecated
```

### Counting and summing

`aggregation` turns the rows or linked entities into one number, which
`operator` and `value` (or `value_of`) then test. This finds requirements
that no verification protocol verifies:

```yaml
from: Requirement
where:
  field: verified_by
  where:
    field: type
    operator: equals
    value: VerificationProtocol
  aggregation: {type: count}
  operator: less_than
  value: 1
```

| `type` | Gives |
| --- | --- |
| `count` | How many rows or linked entities match the `where`. |
| `sum`, `min`, `max`, `avg` | The total, smallest, largest or average of a number `field` of each matching item, e.g. `{type: max, field: row_rating}`. |

The `where` is optional: without it, every row or linked entity counts.
Each item counts once, even when two have equal values. With no items,
`count` and `sum` are 0, and `min`, `max` and `avg` have no value, so a
comparison with them does not hold (`missing` does). These are the rules
of [calculated fields]({{< relref "docs/concepts/calculated-fields" >}})'
`COUNT`, `SUM`, `MIN`, `MAX` and `AVG`.

### Duplicates

`duplicated` holds when another entity of the `from` types has the same
value. It finds copy-paste errors:

```yaml
from: Requirement
where:
  field: title
  operator: duplicated
```

Both entities sharing the value match. A list of fields compares them
together: `field: [title, status]` holds only when another entity has the
same title *and* status. Values compare by type, so the number `2` and the
text `"2"` differ, and an entity with no value never matches. The
comparison is with every entity of the `from` types, whatever else the
`where` asks; the other conditions then filter the result. `duplicated`
compares single values, so it can't be used on a list, a many-link or a
reverse link, or inside a `where` on rows or linked entities.

Outside a `where`, a field with several values (a link with
`cardinality: many`, a reverse link, a list) matches when any of its values
does, and `not_equals` and `not_in` hold only when none does.

Values compare only with values of the same type. A link compares as the
IDs it points to. Dates are written `YYYY-MM-DD`. A value is read by its
YAML type, as in entity files, so a quoted `"2"` is text, not a number.
[Calculated fields]({{< relref "docs/concepts/calculated-fields" >}}) can be
used in conditions and `order_by` like any other field.

### Sorting

`order_by` sorts ascending by each field in turn, then by ID. Entities
without a value sort after those with one. A field with several values
cannot be used.

### Several types

`from` can list several types. Their entities form one list, sorted
together:

```yaml
from: [Risk, Requirement]
order_by: order
```

Each result still renders through its own type's template. A field named
in `where`, `order_by` or `render: field:` must exist on at least one of
the types, with the same kind of value on every type that has it. On an
entity whose type does not have the field, it is empty, as if it were
unset: a test on it does not hold (so `not_equals` does), the entity sorts
after those with a value, and `render: field:` marks it *empty*.

### Rendering

| `render:` | Shows each entity as |
| --- | --- |
| `full` | The whole entity, through its type's template or one the query chooses (see [Custom templates](#custom-templates)), like `![[ID]]`. Headings are nested under the heading the block sits in. |
| `id` | A list of links showing IDs, like `[[ID]]` without a [citation style](#citation-style). |
| `field:title` | A list of that field's values, each linked, like `[[ID#title]]`. |

A list made by `id` or `field:` is not a citation: its entities aren't in
the page's [reference list](#reference-lists).

A query that matches nothing shows *No Requirement matches this query.*

### Custom templates

Entities do not have to render the same way every time they are used. When a
query does not specify a template, the default is derived from the entity's
type name. For example, a `Requirement` type renders through the default
`Requirement` template (`templates/Requirement.tmpl`, or the built-in page if
there is none) unless something else is requested.

A query may override that default to suit a particular context. The override
may apply to all results or, when a query mixes different entity types, to
specific types in the result set. This lets one document show a compact
checklist view of requirements while another shows a detailed risk register,
without creating separate entity types for each presentation.

```yaml
from: [Requirement, Risk]
templates:
  Requirement: requirement-checklist
  Risk: risk-summary
```

If a result's type is not listed in `templates`, it falls back to the default
template for that type. A single-template override can also be written as
`template: requirement-checklist`, which applies to every result whatever its
type, typically in a query that selects one type.

The templates a query names are files in `templates/`, called
`templates/<name>.tmpl`, with a lower-case, hyphenated name such as
`requirement-checklist.tmpl`. That keeps them apart from type templates
(`Requirement.tmpl`) and the site layout (`_layout.tmpl`). A named template
sees the same fields as the type's own template and is used only where a
query asks for it, never for the entity's own page. `template` and
`templates` apply to `render: full`, the mode that renders each result
through a template.

### Errors

A query block that cannot be run stops the export with exit code `2`,
naming the file and line, so a document is never published with a section
silently missing:

```text
export: DOC/DOC-0001.md:29: query block: unknown operator "=" (valid operators: equals, not_equals, greater_or_equal, less_or_equal, greater_than, less_than, exists)
```

The same happens for an unknown type, field or sub-field, a field that none
of the selected types has, a value an enum does not allow, a value of the
wrong type, and `greater_than` or `less_than` used on anything except
numbers, dates and text. For templates, it happens for a template that does
not exist (even when the query matches nothing), a `templates` entry for a
type the query does not select, and `template` and `templates` used
together or with `render: id` or `field:`.

## Other fenced blocks

A `caption` block captions the block before it; see
[Images and captions]({{< relref "docs/concepts/exporting#captions" >}}).
Any other fenced code block is shown as code. Two more languages are
reserved for diagrams, `mermaid` and `drawio`, which Provenance will render.
Until it does, a block in one of them is shown as its source under a note
saying it is not rendered by this version, so it is never mistaken for
content.

`provenance validate` reports a block in a language this version cannot
render yet, wherever it is. A project can
also limit the languages its entities use with a rule over `code_blocks`,
a list every entity has with one row per fenced block (`none` stands for
a block with no language):

```yaml
# rules/block-languages.yaml
severity: error
message: "code block in a language this project does not use"
from: any
where:
  field: code_blocks
  where:
    field: language
    operator: not_in
    value: [query, caption, mermaid, text, none]
```
