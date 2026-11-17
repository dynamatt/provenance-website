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
`DOC/DOC-0006.md` and `DOC/DOC-0007.md`. Its reports go further, with a
section per component: see
[Sections by component](#5-sections-by-component).

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

## 5. Sections by component

A report on a whole system reads best as a section per component and a
subsection per subcomponent, each listing the requirements allocated to
it. Nothing in Provenance groups: the tree is the components' own links,
and the nesting is embedding, whose headings Provenance already nests and
numbers.

Give components a parent, and allocate each requirement to one:

```yaml
# schema/Component.yaml, added
  - name: parent
    type: link
    target: [Component]
    cardinality: one
    reverse_name: subcomponents

# schema/Requirement.yaml, added
  - name: component
    type: link
    target: [Component]
    cardinality: one
    reverse_name: requirements
    required: true
```

The report's scope selects components, here the top of the tree:

```yaml
# schema/VerificationSummaryReport.yaml
  - name: scope
    type: query
    of: [Component]
    reverse_name: reported_in

# DOC/DOC-0006.md
scope:
  from: Component
  where:
    field: parent
    operator: missing
```

A named template, `templates/requirements-of-component.tmpl`, lays out one
component: a heading, its requirements as matrix rows, then its
subcomponents, each through the same template. It reaches them with a
query block it writes and runs with `markdown`, choosing itself again:

```go-html-template
<section class="component-section">
<h1>{{title .}} <span class="id">{{.ID}}</span></h1>
{{- with .requirements}}
<table class="trace-matrix">…one row per requirement, as in step 3…</table>
{{- else}}
<p class="none">No requirement is allocated to {{title .}} itself.</p>
{{- end}}
{{- if .subcomponents}}
{{markdown (printf "```query\nfrom: Component\nwhere:\n  field: parent\n  operator: equals\n  value: %s\ntemplate: requirements-of-component\n```\n" .ID)}}
{{- end}}
</section>
```

The report's template starts the tree the same way, once per component in
its scope:

```go-html-template
{{define "after-body"}}
<section class="report-components">
<h2>Verification by component</h2>
{{- range .scope}}
{{markdown (printf "```query\nfrom: Component\nwhere:\n  field: id\n  operator: equals\n  value: %s\ntemplate: requirements-of-component\n```\n" .ID)}}
{{- end}}
</section>
{{end}}
```

Each component is embedded as a query block's result is, so its `<h1>`
nests under the heading before it: the system is section 1.1, the pulse
generator 1.1.1, its ASIC 1.1.1.1. A component that is its own ancestor is
an embed cycle, which stops the export.

Two things to know:

- **Page history.** A report's page is marked changed by edits to the
  components its query reads, but not by an edit to a requirement shown
  in a section, nor to a protocol or execution in its rows. That is true
  of anything a template reaches by following links.
- **Checking coverage.** A report covers its components and everything
  under them, so the rule checking that every requirement is in a report
  follows the tree: see `rules/requirement-reported.yaml` in the example.

## 6. A report for a build

A verification summary is often written for one build. Give the report the
build it summarises, and its query selects the component that build is
of, reading the build from the report itself:

```yaml
# schema/VerificationSummaryReport.yaml, added
  - name: build
    type: link
    target: [BuildStandard]
    cardinality: one
```

```yaml
# DOC/DOC-0007.md
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

Inside the query, `^` reads the entity holding it, as in any sub-query:
here the condition is on a component's `builds`, so `^` would be the
component and `^^` is the report. The query is the same for every report
about a build; the next build's report changes only `build`. The template
can name the build too, with `{{with .build}}{{link .}} {{.version}}{{end}}`.

## Variations

- **Gathering further.** `collect .scope "verified_by" "tested_by" "build_tested"`
  lists every build the scope's executions ran on, once each.
- **Kinds of a kind.** A type can extend the report type in turn and fill
  another block; each template down the chain is laid over the one above.
- **Without Document.** Any type can be extended this way. A
  `TestReport` extending `VerificationEvidence` keeps the evidence page and
  adds its own sections.
