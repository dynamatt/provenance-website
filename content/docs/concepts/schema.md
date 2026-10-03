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
    required: true
  - name: statement
    type: text
    body: true
    help: "The shall-statement. Ref: ISO 13485 §7.3.3"
  - name: status
    type: ApprovalStatus
    default: draft
```

Every field has a `name` and a `type`. Field names are `snake_case`, and
must be unique within a type. The other field keys are:

| Key | Applies to | Meaning |
| --- | --- | --- |
| `required` | any field | The field must be set. |
| `default` | any field | The value the field takes when it is not set. |
| `help` | any field | Guidance shown to authors. |
| `body` | `text` | Stores this field as the Markdown body below the frontmatter. At most one top-level `text` field per type may set it; a type without one has a freeform body. |
| `target` | `link` | The entity types a link may point to, e.g. `[Requirement, Design]`. |
| `cardinality` | `link` | `one` (a single ID) or `many` (a list of IDs). Required. |
| `reverse_name` | `link` | The name under which the target sees this link, e.g. `implemented_by` for `implements`. |
| `formula` | `calculated` | The expression the value is computed from. See [Calculated fields]({{< relref "docs/concepts/calculated-fields" >}}). |
| `fields` | `list` | The list's row fields, declared inline. |
| `of` | `list` | The type the list holds: a built-in type, an enum or a record. |

`required` and `default` are recorded in the schema now; they will be
enforced and applied when `provenance validate` is implemented.

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

## Errors

Provenance stops with exit code `2` and reports the file and line when the
schema cannot be read, for example when:

- a field names an unknown type;
- a name is declared twice;
- a link has no `cardinality`;
- a list has neither `fields` nor `of`, or has both;
- a record contains itself.

Other problems, such as a link `target` that names no declared type, are
reported by `provenance validate` once it is implemented.
