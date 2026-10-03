---
title: "Exporting the DHF"
description: "Export the design history file as a website, scope it to a document, and trace every page to the commit it came from."
---

`provenance export website` writes the design history file as a static
website: a folder of HTML that works offline, with every entity on its own
page. This page covers what goes into an export: its scope, the provenance
stamped on every page, and how figures are numbered. The
[CLI reference]({{< relref "docs/cli/export" >}}) lists the flags.

## Scope

By default an export contains every entity. `--scope` limits it to part of
the repository, in one of two ways.

**An entity file.** The export contains the entity and everything it pulls
in: the entities it embeds with `![[ID]]`, the results of its
[query blocks]({{< relref "docs/concepts/documents" >}}), and the same for
each entity shown in full inside it. The entity becomes the site's main
page, rendered through its own template. Pointed at a document, this
exports that document:

```bash
provenance export website --scope DOC/DOC-0001.md --out _doc
```

`_doc/index.html` is then the requirements specification, and only the
requirements and figures it shows have pages. Pointed at an entity that
pulls nothing in, the export holds that entity alone.

**A query file.** A YAML file with `from` and an optional `where`, written
like a [query block's]({{< relref "docs/concepts/documents#query-blocks" >}}),
scopes the export to its results, listed on the main page:

```yaml
# scopes/approved-requirements.yaml
from: Requirement
where:
  field: status
  operator: equals
  value: approved
```

Only entities in scope get a page. A reference to an entity outside the
scope keeps its text but is not a link, so it reads as the entity's
citable ID, such as `USR-0001`.

## Provenance on every page

When the repository is a git repository, every export records where it came
from. The built-in page layout shows it in the footer:

- **Commit**: the commit the export was generated from.
- **Content hash**: a SHA-256 hash of the whole design history file as
  committed, such as `sha256:8827af61…`. It ends in `-dirty` when the
  working tree has uncommitted changes, so an export from uncommitted work
  can't pass for a committed one.
- **This page last changed in**: the most recent commit that changed
  anything the page shows: the entity's own file, everything it embeds or
  queries, and the templates and schema that render it. Checking out that
  commit reproduces the page exactly, even if the rest of the repository
  has moved on.

Each page also lists its **revision history**: every commit that changed
what it shows, with date, author, change summary and any release tags.

Outside a git repository the export still works, without these stamps.

### Checking the content hash

`provenance verify content` prints the content hash of the working tree's
commit, with `-dirty` when it has uncommitted changes. Compare it with the
hash printed on an exported page to confirm the page came from that
content. Pass `--commit` to hash an earlier commit or tag, and
`--expected <hash>` to have the command exit with code `1` when the hash
differs:

```bash
provenance verify content --commit v1.0 --expected sha256:8827af61…
```

The hash covers every committed file of the repository except the
signature records in `.signatures/`, which are made about the content and
so can't be part of it. It is computed over the files exactly as git
stores them, so a Windows checkout gives the same hash as any other. The
exact algorithm is part of the product's validated design.

## Numbered figures

Figures, tables and other captioned entities are numbered in each document
in the order they appear, like a word processor's captions. Which entity
types are numbered, and how, is set in `templates/_captions.yaml`:

```yaml
Figure: [Figure]
Table: [DataTable]
```

Each key is a label with its own sequence; it lists the types numbered in
it. In a document that embeds two figures and a table, the figures are
*Figure 1* and *Figure 2* and the table is *Table 1*.

A reference to a numbered entity, `[[FIG-0001]]`, becomes a link to it
within the document showing its number, such as *Figure 1*. This works
even when the reference comes before the figure, and the numbers follow
when figures are reordered. `[[FIG-0001|the control loop]]` keeps its own
text. A reference to a figure the document does not include is an ordinary
link to the figure's page.

A template shows an entity's number with `.CaptionNumber`:

```html
<figure>
{{markdown .Body}}
<figcaption><strong>{{.CaptionNumber}}</strong> {{.Title}}</figcaption>
</figure>
```

The built-in page shows the number and title below a numbered entity.
