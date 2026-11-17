---
title: "Writing a schema"
description: "Declare entity types, enums and records, and the field types they can use."
---

A project's schema lives in the `schema/` folder at the repository root. It
is plain YAML, reviewed and versioned like every other file in the
repository.

```text
schema/
  Requirement.yaml       one file per entity type
  Risk.yaml
  enums/
    ApprovalStatus.yaml  one file per enum
  records/
    Equipment.yaml       one file per record: a shared row shape for lists
```

Only `*.yaml` files directly in each folder are read. Entity types, enums
and records share one namespace: no two declarations may have the same name,
and none may be named after a built-in field type.

## Entity types

Each file in `schema/` declares one entity type, such as a requirement or a
risk: the type's name, its ID prefix and its fields, in the order they are
shown.

```yaml
type: Requirement
id_prefix: REQ
fields:
  - name: title
    type: string
    title: true
    required: true
  - name: statement
    type: text
    body: true
    help: "The shall-statement. Ref: ISO 13485 §7.3.3"
  - name: status
    type: ApprovalStatus
    required: true
```

A type can instead read its entities from a CSV file, one per row, by
declaring a `source:`; its fields are declared the same way. See
[Importing data]({{< relref "docs/concepts/importing-data" >}}).

Every field has a `name` and a `type`. Field names are lower-case
`snake_case`, a letter then letters, digits and single underscores, such as
`verified_by`, and must be unique within a type. A template reads a field by
this name, `.verified_by`. `code_blocks` is taken: it is the list of
fenced blocks every entity has for
[conditions]({{< relref "docs/concepts/documents#what-a-condition-can-read" >}}),
so neither a field nor a `reverse_name` can use it. The other field keys
are:

| Key | Applies to | Meaning |
| --- | --- | --- |
| `required` | any field | The field must be set. |
| `help` | any field | Guidance shown to authors. |
| `friendly_name` | any field | How the built-in pages label the field, such as `Verified by`. Without one, they show its name as written: `verified_by`. |
| `title` | `string` | `true` makes the field the entity's title: shown beside its ID on its page, in link tooltips and on the main page, and given to templates by `{{title .}}`. At most one top-level field per type; a type without one has entities without titles. No field is a title by its name. |
| `body` | `text` | Stores this field as the Markdown body below the frontmatter. At most one top-level `text` field per type may set it; a type without one has a freeform body. |
| `target` | `link` | The entity types a link may point to, e.g. `[Requirement, Design]`. |
| `cardinality` | `link` | `one` (a single ID) or `many` (a list of IDs). Required. |
| `reverse_name` | `link`, `query` | The name under which the target sees this link, e.g. `implemented_by` for `implements`. |
| `formula` | `calculated` | The expression the value is computed from. See [Calculated fields]({{< relref "docs/concepts/calculated-fields" >}}). |
| `fields` | `list` | The list's row fields, declared inline. |
| `of` | `list`, `query` | The type the list holds: a built-in type, an enum or a record. For a query field, the entity types its query may select, e.g. `[Requirement]`. Required. |
| `column` | any field of a type with a `source` | The CSV header the field reads, when it isn't the field's name. |

There are no default values. What an entity's file says is its whole
value: a field the file leaves out is empty everywhere it is read, so a
change to the schema never changes an entity whose file did not change.
A field that must have a value says `required: true`, which
`provenance validate` enforces: a required field that is missing, empty,
or an empty list is an error at the entity's `id:` line.

## Field types

| Type | Value in an entity file |
| --- | --- |
| `string` | A single line of text. |
| `text` | Text, rendered as Markdown. |
| `number` | An integer or decimal number. |
| `date` | A date as `YYYY-MM-DD`. |
| `boolean` | `true` or `false`. |
| `link` | The ID of another entity (`cardinality: one`) or a list of IDs (`cardinality: many`). |
| `list` | A list of rows, or of values of one type. See [Lists](#lists). |
| `calculated` | Nothing: the value is never stored. It is computed from `formula`; see [Calculated fields]({{< relref "docs/concepts/calculated-fields" >}}). |
| `query` | A query: `from`, and `where` and `order_by` if wanted. The value is the entities it selects. See [Query fields](#query-fields). |
| *an enum's name* | One of the enum's values. |

## Enums

An enum is a fixed set of values, declared once in `schema/enums/` and used
by naming it as a field's type.

```yaml
# schema/enums/ApprovalStatus.yaml
enum: ApprovalStatus
values: [draft, in_review, approved, deprecated]
```

Use an enum for a label. When each value needs data of its own, such as a
numeric severity score, make it an entity type instead, and link to it.

## Links

A link is declared once, on the entity type that is written second. For example,
a verification protocol declares that it `verifies` a requirement, so the
requirement does not change when its verification is added. `reverse_name`
names the link as seen from the requirement:

```yaml
# schema/VerificationProtocol.yaml
  - name: verifies
    type: link
    target: [Requirement]
    cardinality: many
    reverse_name: verified_by
```

The requirement then has a derived `verified_by` relationship, listing
every protocol that verifies it. A `reverse_name` must not repeat a field
declared on the target type, because the relationship would then be stored
twice.

## Query fields

A query field holds a query instead of IDs. Its value is the entities the
query selects, worked out at every export and every `validate`, so it
never falls behind them. Use one where an entity says *which* entities it is
about rather than listing them, such as the requirements a report covers.

```yaml
# schema/VerificationSummaryReport.yaml
  - name: scope
    type: query
    of: [Requirement]
    reverse_name: reported_in
```

```yaml
# DOC/DOC-0006.md frontmatter
scope:
  from: Requirement
  where:
    field: status
    operator: equals
    value: approved
  order_by: order
```

The query is written like a
[query block]({{< relref "docs/concepts/documents#query-blocks" >}})'s:
`from`, `where` and `order_by`, with the same conditions. `from` must name
types in the field's `of:`, or types extending them. A query block's
`render` and `template` don't apply: the entity's template shows the
results.

Everywhere else the field reads like a link with `cardinality: many` to
the entities selected, in the query's order:

| Where | Reads |
| --- | --- |
| A template | `.scope`, a list of entities. `queryof . "scope"` gives the query itself. See [Templates]({{< relref "docs/concepts/templates#values" >}}). |
| A condition or rule | `{field: scope, where: …}`, `exists`, `missing`, as for a link. |
| A formula | `COUNT(scope)`, `scope.order`, as for a link. |
| The selected entities | `reported_in`, its `reverse_name`, as for a link. |

### Reading the entity that holds the query

`^field` in the query reads the entity holding it, as in a
[sub-query]({{< relref "docs/concepts/documents#other-entities" >}}). A
report can then name the build it summarises and select the component that
build is of, with a query every such report shares:

```yaml
build: BLD-0002
scope:
  from: Component
  where:
    field: builds
    where:
      field: id
      operator: equals
      value_of: ^^build
```

Each sub-query adds a level: inside the condition on a component's
`builds`, `^` is the component and `^^` the report.

### Queries that read each other

A query may select by a calculated field, and a formula may count a
query's results. A query may also select by the `reverse_name` another
query field feeds. Provenance works out the order. A query that depends on
its own results through `missing`, `not_in`, `not_equals` or a count has no
answer: one report selecting the requirements no report covers, itself
included, is an example. `provenance validate` reports each query in such
a loop, and its value is empty.

A query field can't be a sub-field of a list's rows, nor a field of a type
read from a [CSV file]({{< relref "docs/concepts/importing-data" >}}).

## Lists

A list field holds an ordered list. Use a list for data that belongs to the
entity and is frozen with it, such as the equipment used in a test. Use a
link when the entity refers to something that has its own lifecycle.

A list declares what it holds in one of two ways: `fields` or `of`, never
both.

### Rows declared inline

With `fields`, each item is a row with the given sub-fields. Use this when
only one entity type uses that row shape.

```yaml
# schema/Risk.yaml
  - name: failure_modes
    type: list
    fields:
      - name: description
        type: text
      - name: severity
        type: link
        target: [SeverityLevel]
        cardinality: one
```

```yaml
# RSK/RSK-0001.md frontmatter
failure_modes:
  - description: Sense-electrode lead fracture causes signal loss
    severity: SEV-0003
```

### A list of another type

With `of`, the list names a type declared somewhere else:

- a built-in type: `string`, `text`, `number`, `date` or `boolean`
- an enum
- a record (see below)

A list of a built-in type or an enum holds one plain value per item:

```yaml
# schema/Design.yaml
  - name: standards
    type: list
    of: string

# schema/Requirement.yaml
  - name: verification_methods
    type: list
    of: VerificationMethod   # an enum in schema/enums/
```

```yaml
# a Design's frontmatter
standards: [IEC 60601-1, IEC 62304, ISO 14708-3]
# a Requirement's frontmatter
verification_methods: [test, analysis]
```

A list cannot hold `link`, `list` or `calculated` values. For a list of
links, declare a `link` with `cardinality: many`. A list cannot name an
entity type either: it would copy that type's fields without making a link,
so link to the entities instead.

## Records

A record is a named row shape, declared once in `schema/records/` and used
by any list with `of`. Use one when several entity types record the same
kind of row, such as the equipment used in verification and validation
tests.

```yaml
# schema/records/Equipment.yaml
record: Equipment
fields:
  - name: name
    type: string
  - name: serial_number
    type: string
  - name: calibration_due_date
    type: date
```

```yaml
# schema/VerificationEvidence.yaml
  - name: equipment_used
    type: list
    of: Equipment
```

Records are not field types: `type: Equipment` is an error, and
`type: list` with `of: Equipment` is the correct form. Row data is written
the same way whether the list's fields are inline or come from a record.
A record's fields follow the same rules as a list's inline `fields`. They
may include links and lists, including lists of other records. A record may
not contain itself, directly or through other records.

## Extending a type

A type can extend another with `extends:`. It has every field of the type
it extends, first and in that type's order, then the fields it declares
itself. Use it when several types share fields, such as two kinds of
evidence, or when a type is a kind of another, such as a report that is a
document.

```yaml
# schema/VerificationSummaryReport.yaml
type: VerificationSummaryReport
extends: Document
fields:
  - name: scope
    type: link
    target: [Requirement]
    cardinality: many
    reverse_name: reported_in
```

A verification summary report then has Document's `title` and `status`,
then its `scope`. A type extends one other type, which may itself extend
another; it may not lead back to itself. It adds fields and never changes
its parent's: declaring a field the parent already has is an error.
Calculated fields are inherited like any other, and so are the reverse
links its parent's fields give other types.

### Where a type stands for its subtypes

Wherever a type is named, it stands for itself and every type extending
it:

| Where | Means |
| --- | --- |
| A link's `target: [Document]` | The link may point at a Document or a verification summary report. |
| `from: Document` in a query block or rule | Every Document and every verification summary report. |
| `templates: {Document: …}` in a query block | The template is used for both, unless a subtype is given its own. |
| A reverse link to `Document` | Verification summary reports have it too. |

A condition's `type` is still the entity's own type, so
`{field: type, operator: equals, value: Document}` matches plain Documents
only. To include the types extending one, use
[`is_a`]({{< relref "docs/concepts/documents#conditions" >}}):
`{field: type, operator: is_a, value: Document}`. Templates have the same
test as [`isa`]({{< relref "docs/concepts/templates#functions" >}}).

### IDs

A type extending another numbers its entities in its parent's series,
unless it declares an `id_prefix` of its own. A verification summary
report without one is `DOC-0006`, beside the plain Documents, the way
controlled documents are numbered whatever their kind. Its file's `type:`
says which type it is. Two types may share a prefix only this way, down
one chain of types.

### Abstract types

A type marked `abstract: true` has no entities of its own: it declares
fields for the types extending it, and has no `id_prefix`.

```yaml
# schema/Evidence.yaml
type: Evidence
abstract: true
fields:
  - name: title
    type: string
    title: true
    required: true
  - name: execution_date
    type: date
    required: true
  - name: equipment_used
    type: list
    of: Equipment

# schema/VerificationEvidence.yaml
type: VerificationEvidence
extends: Evidence
id_prefix: EVD
fields:
  - name: tests
    type: link
    target: [VerificationProtocol]
    cardinality: one
```

Named in `from:` or a link's `target:`, an abstract type stands for its
subtypes alone. One rule `from: Evidence` checks verification and
validation evidence alike. An entity whose `type:` is an abstract type is
an error.

### What isn't inherited

Only fields are inherited. `source:`, `pages:` and `abstract:` describe the
type itself, so a type extending another declares its own, and
`id_prefix` is inherited only as described in [IDs](#ids). An entity's
page shows its own type, and the main page lists it under that type.

Because inherited fields come first, moving a field from a type to a type
it extends changes the order of its entities' frontmatter.
[`provenance fmt`]({{< relref "docs/concepts/formatting" >}}) rewrites
them.

## Entities without pages

Each entity has its own page in the export, unless its type says
`pages: false`. The type's entities are then shown together on one page,
`entities/<Type>.html`, and every link to one of them points at its place
there. Use it for a few small entities that read better together, such as
severity levels or glossary terms, and for a large
[CSV file]({{< relref "docs/concepts/importing-data" >}}) of near-identical
rows.

```yaml
# schema/SeverityLevel.yaml
type: SeverityLevel
id_prefix: SEV
pages: false
fields:
  - name: label
    type: string
  - name: score
    type: number
```

A [`templates/_<Type>.tmpl`]({{< relref "docs/concepts/templates#type-pages" >}})
lays the page out; without one, it is a table, or each entity in turn
when they have bodies. Other types have no type page: to list a type,
write a [Document]({{< relref "docs/concepts/documents" >}}) with a query
block.

## Errors

Provenance stops with exit code `2` and reports the file and line when the
schema cannot be read, for example when:

- a field names an unknown type;
- a name is declared twice;
- a link has no `cardinality`;
- a field's `name`, or a link's `reverse_name`, is not lower-case
  `snake_case`;
- `pages` is not `true` or `false`;
- `title: true` is on a field that isn't a top-level `string`, or on two
  fields of one type;
- a list has neither `fields` nor `of`, or has both;
- a record contains itself;
- `extends:` names no type, or leads back to the type;
- a type declares a field the type it extends already has;
- an abstract type has an `id_prefix` or a `source:`, or `abstract` is
  not `true` or `false`;
- a query field has no `of:`, is in a list's rows, or is on a type with a
  `source:`.

Other problems don't stop the schema loading; `provenance validate`
reports each of them at its file and line:

- a key the schema format doesn't have, such as `default`, or one that
  doesn't apply to the field, such as `target` on a `string`;
- a link `target` that names no declared type;
- a formula that doesn't parse, names a field that doesn't exist, or
  depends on itself;
- a type without an `id_prefix`, or two types sharing one other than
  down one [chain of types](#ids);
- and, as warnings, an enum or record nothing uses, and an entity type
  with no entities and no links to or from it.

`provenance validate` also checks every entity against its type: fields
the type doesn't declare, values of the wrong type or outside their enum,
calculated fields written in the file, unknown sub-fields in list rows,
entities of an [abstract type](#abstract-types), [query fields](#query-fields)
whose query can't run (`schema/query`, at its line in the query), and IDs that aren't the type's `id_prefix`, a hyphen and at least four
digits.
