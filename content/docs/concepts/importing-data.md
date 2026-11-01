---
title: "Importing data"
description: "Load entities from a CSV file: dumps from another system, such as complaints or an issue tracker's anomalies, and tables your team keeps by hand."
# Planned for v0.5.0-alpha (Epic 6 in PLAN.md). Published at the epic's
# close-out, once checked against what was built.
draft: true
---

Some records are kept in another system, such as the complaint-handling
process, an issue tracker or a test runner. Others are tables by nature.
Provenance can load these records straight from a CSV file in the
repository. Each row becomes an entity like any Markdown one: it has an
ID, links and is linked to, is selected by queries and rules, has a page
in the export, and counts in the content hash.

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

A type is backed by a file when its schema has a `source:`. Its fields are
declared as for any type; `column:` names the header when it differs from
the field's name.

```yaml
# schema/Complaint.yaml
type: Complaint
id_prefix: CMP
source:
  csv: data/complaints.csv             # the file, from the repository root
  system: Complaint handling (QMS-CH)  # where every dump comes from
  id: complaint_no                     # the column holding each row's ID
fields:
  - name: received
    type: date
    column: Date Received
  - name: product
    type: string
  - name: failure_mode
    type: string
  - name: harm_occurred
    type: boolean
  - name: summary
    type: text
```

```text
complaint_no,Date Received,product,failure_mode,harm_occurred,summary
CMP-2026-0141,2026-08-14,Lead kit,"Lead fracture, distal",false,Signal lost after a fall.
```

A Risk then links to its complaints, and a calculated field counts them:

```yaml
# schema/Risk.yaml
  - name: complaints
    type: link
    target: [Complaint]
    cardinality: many
    reverse_name: cited_by_risks
```

The source's keys:

| Key | Meaning |
| --- | --- |
| `csv` | The file, as a path from the repository root. |
| `system` | The system the file is exported from. Leave it out for a [table kept by hand](#tables-kept-by-hand). |
| `id` | The column holding each row's ID. |
| `delimiter` | The character between cells. Default `,`. |
| `separator` | The character between the IDs or items in one cell. Default `;`. |
| `pages` | `false` for no page per row, for a large file. Default `true`. |

The file is RFC 4180 CSV in UTF-8 with a header row, which is what
spreadsheets and most systems export. A column the schema doesn't name is
ignored, and `validate` warns of it, so a column renamed in the source
system is noticed.

## How a cell is read

A cell takes the same field types, written the same way, as a value in an
entity file. There are no CSV-only types: a value is valid in a cell
exactly when it is valid in a file, so a rule, query or template reads a
row's fields as it reads any entity's.

| Field type | In a cell |
| --- | --- |
| `string` | One line of text. |
| `text` | Markdown, wikilinks included. |
| `number` | `2.5`, `-3`, `1000`. No thousands separators, units or `%`. |
| `date` | `YYYY-MM-DD`. |
| `boolean` | `true` or `false`. |
| *an enum* | One of its values. |
| `link`, `cardinality: one` | One ID. |
| `link`, `cardinality: many` | IDs separated by the source's `separator`, such as `REQ-0012;REQ-0031`. |
| `list` of a built-in type or an enum | Items separated by the source's `separator`. |
| `list` of rows | Not available: a row's fields are its columns. |
| `calculated` | Never a column; computed as on any entity. |

An empty cell is a field with no value, the same as a field an entity file
leaves out, so `required: true` works the same way. A system that writes
dates another way, such as `14/08/2026`, needs to be set to export ISO
dates, or its file converted before it is committed.

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
  system: Issue tracker (Jira)
  id: Key
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
  - name: resolved
    type: boolean
```

```text
Key,Summary,Requirements,Resolved
BUG-1042,Alarm tone clipped at max volume,REQ-0012;REQ-0031,false
```

Each requirement then lists its `anomalies`, and a rule can stop a release
while a requirement has an unresolved one. A link to an ID that doesn't
exist is reported at the row's line, as it would be in an entity file.

## Dumps and their sidecar

A copied file doesn't say when it was exported, so a type with a `system:`
needs a sidecar beside each dump that does:

```yaml
# data/complaints.csv.yaml, rewritten with each dump
exported: 2026-09-30
sha256: 3b5c7e…    # sha256sum data/complaints.csv
```

`validate` fails when the sidecar is missing, or when its hash isn't the
file's, so a new dump can't be merged without saying when it was exported.
Git records when the copy entered the design history file and who
committed it. Each row's page shows all of it.

To bring in a new dump, replace the file, rewrite the sidecar and commit
both. A row's history moves only when that row changed, so a monthly dump
doesn't make every complaint look edited.

## Tables kept by hand

A type whose `source:` has no `system:` is a table your team keeps in the
repository, such as a standards register. It needs no sidecar, and its IDs
take the `<id_prefix>-NNNN` form of any entity you write (`STD-0001`),
where a dump's IDs are the source system's, used as written. Edit it in a
spreadsheet or a text editor; its pull request's diff shows the rows that
changed.

## What Provenance doesn't do with rows

The CSV file is a copy of another record, so Provenance's commands never
write it: `fmt`, `rename`, the editor and `sign` leave it as it is. A row
has no Markdown body. Formats that need code, such as an eQMS's API or an
XLSX workbook, are for import plugins.
