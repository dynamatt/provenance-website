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
- Each report is a file holding its prose and the requirements it covers.
  Changing the template changes every report at the next export.

The [example repository](https://github.com/dynamatt/provenance-example)
implements this recipe: see `schema/VerificationSummaryReport.yaml`,
`templates/VerificationSummaryReport.tmpl`, `templates/Document.tmpl` and
`DOC/DOC-0006.md`.

## 1. A type that extends Document

Add `schema/VerificationSummaryReport.yaml`:

```yaml
type: VerificationSummaryReport
extends: Document
fields:
  - name: scope
    type: link
    target: [Requirement]
    cardinality: many
    reverse_name: reported_in
    required: true
```

The report has Document's fields first, `title` and `status`, then its
`scope`. With no `id_prefix` of its own it numbers in Document's series.
`reverse_name` gives every requirement a `reported_in` list of the reports
covering it.

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

Add `templates/VerificationSummaryReport.tmpl`, holding only the block it
fills:

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

Because the file holds nothing but a `{{define}}`, it is laid over
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
  - REQ-0001
  - REQ-0002
  - REQ-0003
---
This report summarises the verification of the amplitude control
requirements. The procedures are in [[DOC-0004]] and their executions
are reported in [[DOC-0005]].
```

A second report is a second file with its own `scope`. A requirement with
no protocol shows *Not verified by any protocol* in the matrix, so a gap
in coverage is visible in the report rather than left out of it.

## Variations

- **More than one block.** Fill `before-body` too, for a summary above the
  prose, such as how many requirements the report covers:
  `{{len .scope}}`.
- **Kinds of a kind.** A type can extend the report type in turn and fill
  another block; each template down the chain is laid over the one above.
- **Without Document.** Any type can be extended this way. A
  `TestReport` extending `VerificationEvidence` keeps the evidence page and
  adds its own sections.
