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

An ID that no entity has is shown marked *unresolved*; the export still
succeeds. An entity that embeds itself, directly or through other embeds,
stops the export, because the document would never end.

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

A condition names a `field`, an `operator` and, for every operator except
`exists`, a `value`. Operators are words, not symbols:

| Operator | Holds when the field… |
| --- | --- |
| `equals` | has the value. |
| `not_equals` | does not have the value. This includes a field with no value at all. |
| `greater_than`, `less_than` | is above, or below, the value. |
| `greater_or_equal`, `less_or_equal` | is at least, or at most, the value. |
| `exists` | has at least one value. Takes no `value`. |

A list of conditions means all of them must hold. `any_of` means at least
one must hold:

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

Each alternative in `any_of` is a condition or a list of conditions.
`any_of` cannot contain another `any_of`.

### What a condition can read

| `field:` | Reads |
| --- | --- |
| `status` | A field of the entity. |
| `verified_by` | A reverse link: the entities that link to this one through that `reverse_name`. |
| `id`, `type` | The entity's ID and type. |
| `{list: equipment_used, subfield: calibration_due_date}` | A sub-field of the entity's list rows. |
| `{via: implements, field: status}` | A field of the entities this one links to, through a link or a reverse link. |

`value` is a literal, or `{field: …}` to compare with another field of the
same entity:

```yaml
where:
  field: {list: equipment_used, subfield: calibration_due_date}
  operator: less_than
  value: {field: execution_date}
```

Some fields have several values: a link with `cardinality: many`, a reverse
link, a list, or a field read `{via: …}` from several linked entities. Such
a field matches when any of its values does, and `not_equals` holds only
when none does. Several
conditions on the same list in one list of conditions refer to the same
row: the example below finds equipment that is both serial `A` and overdue,
not one row of each.

```yaml
where:
  - field: {list: equipment_used, subfield: serial_number}
    operator: equals
    value: A
  - field: {list: equipment_used, subfield: calibration_due_date}
    operator: less_than
    value: {field: execution_date}
```

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
| `id` | A list of links showing IDs, like `[[ID]]`. |
| `field:title` | A list of that field's values, each linked, like `[[ID#title]]`. |

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

Any other fenced code block is shown as code. Two more languages are
reserved for diagrams, `mermaid` and `drawio`, which Provenance will render.
Until it does, a block in one of them is shown as its source under a note
saying it is not rendered by this version, so it is never mistaken for
content.

A project can limit the languages its entities use with a Block Language
rule, which `provenance validate` will check once it is implemented. It
reports blocks in a language outside the project's list, and blocks in a
language this version of Provenance cannot render yet, even if listed.
`none` stands for a block with no language.

```yaml
# rules/block-languages.yaml
id: block-languages
rule: BlockLanguage
severity: error
message: "code block in a language this project does not use, or that this version cannot render"
entity_type: any
allowed: [query, mermaid, text, none]
```
