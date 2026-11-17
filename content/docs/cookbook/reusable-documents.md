---
title: "Reusable documents"
description: "Write a kind of document once, as a type and a template, and each document of that kind holds only its prose and what it covers."
---

Some documents are written many times over: a verification summary report
for each subsystem, a test report for each build. Each has the same
sections and the same tables, and differs only in what it covers. Written
as plain Documents, each copies the last one's tables and query blocks, and
the copies drift apart. In this recipe the kind of document is written
once:

- A *Verification Summary Report* type
  [extends]({{< relref "docs/concepts/schema#extending-a-type" >}})
  Document, so each report is a Document: numbered `DOC-`, listed with the
  other documents, cited like them.
- The report's template adds a traceability matrix to the Document
  layout: each requirement the report covers, the protocols that verify
  it, their executions and results.
- Each report is a file holding its prose and a query saying which
  requirements it covers. A requirement approved later joins the report
  at the next export without editing it, and changing the template
  changes every report.

The [example repository](https://github.com/dynamatt/provenance-example)
implements this recipe: see `schema/VerificationSummaryReport.yaml`,
`templates/VerificationSummaryReport.tmpl`, `templates/Document.tmpl`,
`DOC/DOC-0006.md` and `DOC/DOC-0007.md`.

## 1. A type that extends Document

Add `schema/VerificationSummaryReport.yaml`:

```yaml
type: VerificationSummaryReport
extends: Document
fields:
  - name: scope
    type: query
    of: [Requirement]
    reverse_name: reported_in
    required: true
  - name: requirement_count
    type: calculated
    formula: COUNT(scope)
```

The report has Document's fields first, `title` and `status`, then its
own. With no `id_prefix` of its own it numbers in Document's series.
`scope` is a [query field]({{< relref "docs/concepts/schema#query-fields" >}}):
each report writes a query, and the field holds the requirements it
selects. `reverse_name` gives every requirement a `reported_in` list of the
reports covering it, and `requirement_count` counts the scope.

## 2. Blocks in the Document template

In `templates/Document.tmpl`, mark the places a kind of document may add
to, with empty blocks:

```go-html-template
<article class="document" id="{{.ID}}">
<h1><span class="id">{{.ID}}</span> {{title .}}</h1>
<p class="document-meta"><span class="status status-{{.status}}">{{.status}}</span></p>
{{- block "before-body" .}}{{end}}
<div class="body">
{{markdown .Body}}</div>
{{- block "after-body" .}}{{end}}
…
```

A plain Document fills neither, so it renders as before. Without a
`Document.tmpl` of your own, skip this step: the built-in entity page
declares the same two blocks.

If the template lists the documents a page cites, test with `isa` rather
than comparing the type, so reports are listed too:

```go-html-template
{{- range .Citations}}{{if isa . "Document"}}
<li>{{link .}}: {{title .}}</li>
{{- end}}{{end}}
```

## 3. The report's template

Add `templates/VerificationSummaryReport.tmpl`, holding only the blocks it
fills. Before the prose, the report states its scope:

```go-html-template
{{define "before-body"}}
<section class="report-scope">
<p>This report covers {{.requirement_count}} requirements, selected by:</p>
<pre class="query"><code>{{queryof . "scope"}}</code></pre>
</section>
{{end}}
```

After it, the matrix: `.scope` is the requirements the query selected,
each with its own fields and links:

```go-html-template
{{define "after-body"}}
<section class="trace-matrix">
<h2>Traceability matrix</h2>
<table>
<thead><tr><th>Requirement</th><th>Protocol</th><th>Execution</th><th>Result</th></tr></thead>
<tbody>
{{- range .scope}}
{{- $req := .}}
{{- range .verified_by}}
{{- $protocol := .}}
{{- range .tested_by}}
<tr><td>{{link $req}}</td><td>{{link $protocol}}</td><td>{{link .}}</td><td>{{.result}}</td></tr>
{{- else}}
<tr><td>{{link $req}}</td><td>{{link $protocol}}</td><td colspan="2">Not yet executed</td></tr>
{{- end}}
{{- else}}
<tr><td>{{link $req}}</td><td colspan="3">Not verified by any protocol</td></tr>
{{- end}}
{{- end}}
</tbody>
</table>
</section>
{{end}}
```

Then the equipment, each piece once however many protocols in scope need
it. `collect` follows `verified_by` from every requirement, then
`equipment_needed` from every protocol reached:

```go-html-template
<h2>Test equipment</h2>
<ul>
{{- range collect .scope "verified_by" "equipment_needed"}}
<li>{{link .}}: {{title .}}</li>
{{- end}}
</ul>
```

Because the file holds nothing but `{{define}}`s, it is laid over
`Document.tmpl` rather than replacing it (see
[Types that extend another]({{< relref "docs/concepts/templates#types-that-extend-another" >}})).
The report keeps the Document heading, status, body and reference list,
with its matrix after the body.

## 4. One file per report

```markdown
---
id: DOC-0006
type: VerificationSummaryReport
title: Verification Summary Report — Amplitude Control
status: draft
scope:
  from: Requirement
  where:
    field: status
    operator: equals
    value: approved
  order_by: order
---
This report summarises the verification of the approved requirements.
The procedures are in [[DOC-0004]] and their executions are reported in
[[DOC-0005]].
```

A second report is a second file with its own query. A requirement with
no protocol shows *Not verified by any protocol* in the matrix, so a gap
in coverage is visible in the report rather than left out of it.

## 5. One query for every report about a requirement

A report about one requirement covers it and the requirements refining
it. Add a `covers` link to the type, and the query reads it from the
report with `^covers`, so every such report has the same query:

```yaml
# schema/VerificationSummaryReport.yaml, added
  - name: covers
    type: link
    target: [Requirement]
    cardinality: one
```

```markdown
---
id: DOC-0007
type: VerificationSummaryReport
title: Verification Summary Report — Single-Fault Amplitude Ceiling
status: draft
covers: REQ-0002
scope:
  from: Requirement
  where:
    any_of:
      - field: id
        operator: equals
        value_of: ^covers
      - field: parent_requirement
        operator: equals
        value_of: ^covers
  order_by: order
---
```

A rule can then check that every requirement is in some report, reading
`reported_in`:

```yaml
# rules/requirement-reported.yaml
severity: warning
message: "requirement is in no verification summary report"
from: Requirement
where:
  field: reported_in
  operator: missing
```

## Variations

- **Gathering further.** `collect .scope "verified_by" "tested_by" "build_tested"`
  lists every build the scope's executions ran on, once each.
- **Kinds of a kind.** A type can extend the report type in turn and fill
  another block; each template down the chain is laid over the one above.
- **Without Document.** Any type can be extended this way. A
  `TestReport` extending `VerificationEvidence` keeps the evidence page and
  adds its own sections.
