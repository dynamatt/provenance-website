---
title: "Glossary"
description: "Define each term once, cite it inline, and list the terms a document uses in a Definitions section at its end."
---

A design history file uses the same terms in many documents: *ECAP*,
*single fault condition*, *essential performance*. Written out in each
document, their definitions drift apart. In this recipe each term is an
entity, defined once:

- A document cites a term inline with `[[DEF-0001]]`, which shows the term
  linked to its definition.
- Each document ends with a *Definitions* section listing the terms it
  cites, so a reader has the definitions without leaving the page.
- Changing a definition changes it in every document at the next export.
  Its revision history shows who changed it and when.

The [example repository](https://github.com/dynamatt/provenance-example)
implements this recipe: see `schema/Definition.yaml`, `DEF/`,
`templates/_cite.tmpl`, `templates/Document.tmpl` and `DOC/DOC-0001.md`.

## 1. A Definition type

Add `schema/Definition.yaml`:

```yaml
type: Definition
id_prefix: DEF
fields:
  - name: title
    type: string
    required: true
    help: "The term, written out in full."
  - name: abbreviation
    type: string
    help: "Shown in place of the term where the document cites it, e.g. ECAP."
  - name: definition
    type: text
    body: true
    required: true
  - name: source
    type: link
    target: [Reference]
    cardinality: one
    help: "The standard or paper the definition is taken from, if any."
```

The term is the `title`, so a definition reads like any other entity: on
the site's main page, in link tooltips and in query results. `source`
links to a `Reference` entity, so a definition taken from a standard says
which one. Leave `source` out if your schema has no Reference type.

## 2. One file per term

Each term is a file in `DEF/`, with its definition as the body:

```markdown
---
id: DEF-0001
type: Definition
title: Evoked compound action potential
abbreviation: ECAP
source: REF-0003
---
The summed electrical response of the nerve fibres a stimulation pulse
recruits, recorded at a sense electrode shortly after the pulse. Its
amplitude measures how strongly the pulse activated the nerve.
```

## 3. Cite terms inline

In any document, cite a term as you would any entity:

```markdown
The controller regulates on the evoked response ([[DEF-0001]]), which
makes this [[DEF-0002|closed-loop stimulation]].
```

Without a template, a citation shows the ID, `DEF-0001`. To show the
term instead, give `Definition` its own branch in `templates/_cite.tmpl`
(see [Citation style]({{< relref "docs/concepts/documents#citation-style" >}})).
Keep the branches you already have, such as the one numbering References:

```go-html-template
{{- if eq .Type "Definition" -}}
{{link . (or .CitationLabel .Abbreviation .Title)}}
{{- else if .CitationLabel -}}
{{link . .CitationLabel}}
{{- else -}}
{{link .}}
{{- end}}
```

`[[DEF-0001]]` now shows **ECAP**, the abbreviation, linked to the
definition's page with the full term as its tooltip. A term without an
abbreviation shows its title. A label overrides both, so the sentence can
use the wording it needs: `[[DEF-0002|closed-loop stimulation]]`.

Inside a table cell, escape the `|`, because a bare `|` ends the cell:

```markdown
| System | 8 mA under any [[DEF-0003\|single fault]] | [[REQ-0002]] |
```

## 4. A Definitions section in each document

`.Citations` in `templates/Document.tmpl` is every entity the page cites,
embedded content included, in the order the page first cites them. List
the Definitions among them before the rest of the document's end matter:

```go-html-template
{{- $definitions := false}}
{{- range .Citations}}{{if eq .Type "Definition"}}{{$definitions = true}}{{end}}{{end}}
{{- if $definitions}}
<section class="definitions">
<h2>Definitions</h2>
<dl>
{{- range .Citations}}{{if eq .Type "Definition"}}
<dt id="def-{{.ID}}">{{.Title}}{{with .Abbreviation}} ({{.}}){{end}}</dt>
<dd>{{markdown .Definition}}{{with .Source}}<p class="definition-source">Source: {{.Title}} ({{link .}})</p>{{end}}</dd>
{{- end}}{{end}}
</dl>
</section>
{{- end}}
```

The section appears only on a document that cites at least one term.
Like the reference list, it appears once, on the document's own page: when
the document is embedded in another or shown by a query, `.Citations` is
empty. Style it in `templates/style.css`, for example:

```css
.definitions dt { font-weight: 600; margin-top: 12px; }
.definitions dd { margin: 4px 0 0 1.2em; }
.definition-source { color: var(--muted); font-size: 0.85rem; }
```

## Variations

**The whole glossary in one document.** A query block lists every
definition, sorted by term, whatever cites it:

````markdown
## Glossary

```query
from: Definition
order_by: title
```
````

**Order.** The Definitions section lists terms in the order the document
first uses them, as the reference list does. Templates can't sort, so
for an alphabetical list use the query block above.

**First use.** Every citation of a term renders the same way; a citation
can't tell whether it is the term's first on the page. Write the first use
out by hand: `the evoked compound action potential ([[DEF-0001]])`.
