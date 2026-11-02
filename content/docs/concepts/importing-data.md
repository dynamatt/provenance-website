---
title: "Importing data"
description: "Load entities from a CSV file: dumps from another system, such as complaints or an issue tracker's anomalies, and tables your team keeps by hand."
---

Some records are kept in another system, such as the complaint-handling
process, an issue tracker or a test runner. Others are tables by nature.
Provenance can load these records straight from a CSV file in the
repository. Each row becomes an entity like any Markdown one: it has an
ID, links and is linked to, is cited, is selected by queries and rules,
has a page in the export, and counts in the content hash.

## What teams import

| Records | Usually comes from | How it links |
| --- | --- | --- |
| Complaints | Complaint handling (ISO 13485 §8.2.2) | A Risk links to the complaints behind its occurrence rating |
| CAPAs, nonconformances | eQMS | A Risk or Change Request links to them |
| Adverse-event and vigilance reports | Regulatory reporting (21 CFR 803, EU MDR Art. 87) | A Risk links to them |
| Service and returns records | Field service, CRM | A Risk links to them |
| Known anomalies | Issue tracker (IEC 62304 §5.8.3, §9) | Each anomaly links to the requirements it affects |
| Verification results | CI test runner | Each result links to its Verification Protocol |
| SOUP and SBOM components | Dependency scanner (IEC 62304 §8.1.2) | Each component links to the Design that uses it |
| Vulnerabilities | Vulnerability scanner (FDA §524B) | Each vulnerability links to its component |
| Use errors | Formative usability studies (IEC 62366-1) | A Risk links to them |
| Standards register, GSPR checklist | Kept by the team | Designs and requirements link to the rows |

A link is declared on the record written second, as for any
[link]({{< relref "docs/concepts/schema#links" >}}): a risk analysis is
written after the complaints it cites, and an anomaly after the
requirement it affects.

## Declaring a CSV-backed type

A type is read from a file when its schema has a `source:`. Its fields
are declared as for any type; `column:` names the header when it differs
from the field's name.

```yaml
# schema/Complaint.yaml
type: Complaint
id_prefix: CPL
source:
  csv: data/complaints.csv   # the file, from the repository root
  id: complaint_no           # the column holding each row's ID
fields:
  - name: title
    type: string
    column: Subject
  - name: received
    type: date
    column: Date Received
  - name: exported
    type: date
    column: Exported
    required: true
  - name: product
    type: string
  - name: harm_occurred
    type: boolean
  - name: summary
    type: text
```

```text
complaint_no,Date Received,Exported,Subject,product,harm_occurred,summary
CPL-2026-0141,2026-08-14,2026-09-30,Signal loss when bending,"NP-100 lead kit, long",false,Lead fracture at the junction.
```

A Risk then links to its complaints, and a calculated field counts them:

```yaml
# schema/Risk.yaml
  - name: complaints
    type: link
    target: [Complaint]
    cardinality: many
    reverse_name: cited_by_risks
  - name: complaint_count
    type: calculated
    formula: "COUNT(complaints)"
```

The source's keys:

| Key | Meaning |
| --- | --- |
| `csv` | The file, as a path from the repository root. Required. |
| `id` | The column holding each row's ID. Required. |
| `delimiter` | The character between cells. Default `,`. |
| `separator` | What separates the IDs or items in one cell. Default `;`. |
| `pages` | `false` for no page per row, for a large file: the rows are shown on the [table page](#pages). Default `true`. |

The file is RFC 4180 CSV in UTF-8 with a header row, which is what
spreadsheets and most systems export; a byte-order mark and CRLF line
endings are fine. A cell holding the delimiter, a quote or a line break is
quoted, with each quote doubled.

A type's fields can't include a list of rows, which has no form in one
cell, or a `body: true` field: a row has no Markdown body. A calculated
field is computed as on any entity, and never read from a column.

## How a cell is read

A cell takes the same field types, written the same way, as a value in an
entity file. There are no CSV-only types: a value is valid in a cell
exactly when it is valid in a file, and means the same, so a rule, query
or template reads a row's fields as it reads any entity's.

| Field type | In a cell |
| --- | --- |
| `string` | Text. |
| `text` | Markdown, wikilinks and embeds included. |
| `number` | `2.5`, `-3`, `1000`, `1e3`. No thousands separators, units or `%`. |
| `date` | `YYYY-MM-DD`. |
| `boolean` | `true` or `false`. |
| *an enum* | One of its values. |
| `link`, `cardinality: one` | One ID. |
| `link`, `cardinality: many` | IDs separated by the source's `separator`, such as `REQ-0012;REQ-0031`. |
| `list` of a built-in type or an enum | Items separated by the source's `separator`. |
| `calculated` | Never a column; computed as on any entity. |

Spaces around each ID or item are ignored, and an empty one, such as
after a trailing `;`, is left out. An empty cell is a field with no value,
the same as a field an entity file leaves out, so `required: true` works
the same way.

A system that writes dates another way, such as `14/08/2026`, or numbers
with a thousands separator, needs to be set to export ISO dates and plain
numbers, or its file converted before it is committed.

## Rows that link out

A row links to other entities through a `link` field, declared as on any
type. An issue tracker's known anomalies, each naming the requirements it
affects:

```yaml
# schema/Anomaly.yaml
type: Anomaly
id_prefix: BUG
source:
  csv: data/anomalies.csv
  id: Key
  pages: false
fields:
  - name: title
    type: string
    column: Summary
  - name: affects
    type: link
    target: [Requirement]
    cardinality: many
    reverse_name: anomalies
    column: Requirements
  - name: fixed
    type: boolean
    column: Resolved
```

```text
Key,Summary,Requirements,Resolved
BUG-1042,Alarm tone clipped at maximum volume,REQ-0001;REQ-0003,false
```

Each requirement then has its `anomalies`, and a rule can stop a release
while a requirement has one that isn't fixed. A link to an ID that doesn't
exist is reported at the row's line, as it would be in an entity file.

A field can't be named `resolved`, `file` or another name the
[template data]({{< relref "docs/concepts/templates#type-templates" >}})
already uses; map the column to another name with `column:`.

## IDs

A row keeps the ID written in its ID column, so a record can be traced
back to the system that gave it: each must start with the type's
`id_prefix` and a hyphen, such as `CPL-2026-0141` or `STD-0001`. Every ID
is unique across the repository, rows and files alike.

## Where a dump came from

Git records when a file entered the design history file and who
committed it. Anything else you need to record about a dump, such as the
date it was exported or the system it came from, is a field like any
other: declare it, and the export carries it as a column. The complaints
above declare `exported` as a required `date`, so `provenance validate`
fails for a dump without it, and a rule or template can read it.

To bring in a new dump, replace the file and commit it. A per-row export
date changes every row, so each new dump moves every row's history; leave
it out when that matters more than the date.

## Tables kept by hand

A table your team keeps in the repository, such as a standards register,
is read the same way. Edit it in a spreadsheet or a text editor; its pull
request's diff shows the rows that changed.

## Pages

Each row has its own page, `entities/<ID>.html`, which says which file it
is a row of. Each file also has a table page, `entities/<Type>.html`: the
file and its rows, with the file's commits as its revision history. The site's main page links each CSV-backed type to its
table page.

With `pages: false`, rows have no pages of their own. The table page then
shows every row in full, one column per field, and every link or citation
of a row points at its row there.

## In templates

A row is an entity, so a template reads it as any other: its fields, its
`.Type`, its `.LastChangedSHA` and `.Revisions`. Every entity also has
`.File`, the file it is written in, with the same keys for a Markdown
file and a CSV row:

| Accessor | Markdown entity | CSV row |
| --- | --- | --- |
| `.File.Path` | `REQ/REQ-0001.md` | `data/complaints.csv` |
| `.File.Line` | The line of its `id:` | The row's line |

```go-html-template
<p>{{.File.Path}}:{{.File.Line}}</p>
```

`link` and `href` point a row without a page of its own at its row on the
table page. On the
[main page]({{< relref "docs/concepts/templates#main-page" >}}), each type
group read from a file has the file's `.File`, its `.Path` alone.

## History

A row changes when one of the cells its fields read changes, so a dump
that reorders columns or adds one changes no row. A new dump moves the
*last changed* commit of the rows it changed and of the table page, not
every row's.

## What validate reports

| Check | Severity | Reports |
| --- | --- | --- |
| `content/csv-row` | error | A missing or unreadable file, a header naming a column twice or without the ID column, a row with the wrong number of cells or no ID, a quote out of place. |
| `content/csv-column` | warning | A column no field reads, and a field whose column is missing, so a column renamed in the source system is noticed. |

A cell's value, enum, link or ID is checked by the same check as in an
entity file, at the row's line: `schema/invalid-value`,
`schema/enum-value`, `links/unresolved`, `schema/id-format`.

## What Provenance doesn't do with rows

The CSV file is a copy of another record, or a table you edit yourself,
so Provenance's commands never write it: `fmt` leaves it as it is.
Formats that need code, such as an eQMS's API or an XLSX workbook, are
for import plugins.
