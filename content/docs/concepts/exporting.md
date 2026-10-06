---
title: "Exporting the DHF"
description: "Export the design history file as a website, scope it to a document, and trace every page to the commit it came from."
---

`provenance export website` writes the design history file as a static
website: a folder of HTML that works offline, with every entity on its own
page. This page covers what goes into an export: its scope, the provenance
stamped on every page, and images and their captions. The
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
requirements it shows have pages. Pointed at an entity that
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
  anything the page shows. That is the entity's own file; everything it
  embeds or queries; the entities its calculated fields read, such as the
  severity levels behind a risk rating; its images; the entities it cites,
  when its templates list its [references]({{< relref "docs/concepts/documents#reference-lists" >}});
  and the schema and templates that render it. Checking out that commit reproduces the page's
  content, even if the rest of the repository has moved on. The
  stylesheet, and schema or templates the page does not use, do not
  count.

Each page also lists its **revision history**: every commit that changed
what it shows, by the same rule, with date, author, change summary and any
release tags.

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

## Images and captions

### Images

An image in an entity's Markdown is a file in the repository, written as a
path relative to the entity's file, as in any Markdown viewer:

```markdown
![Closed-loop amplitude control](../assets/control-loop.svg)
```

A path starting with `/` is relative to the repository root. The export
copies each image it uses into the site, so the site still works offline.
SVG, PNG, JPEG, GIF and WebP files are supported. A web address, a path
outside the repository or a missing file stops the export with exit code
`2` at the line that uses it.

### Captions

Figures, tables and equations are numbered in each document in the order
they appear, like a word processor's captions. A caption is written where
the figure is used, so the same image can carry a different caption in
each document. It is a `caption` block placed right after the image, table,
embed or diagram it captions:

````markdown
![Closed-loop amplitude control](../assets/control-loop.svg)

```caption
kind: figure
id: control-loop
text: The blocks of the control loop, *as built*.
```
````

| Key | Meaning |
| --- | --- |
| `kind` | `figure`, `table` or `equation`. Required. |
| `id` | A name for references to this caption. Letters, digits, `-` and `_`. |
| `text` | The caption, in Markdown. |

The block must come right after what it captions: a paragraph holding only
an image, a table, an embedded entity (`![[ID]]`), a query block or other
fenced block, or a block of HTML. Anywhere else the export stops with exit
code `2`, so a caption is never attached to the wrong thing.

A reference `[[#control-loop]]` becomes a link to the caption showing its
number, such as *Figure 1*. This works even when the reference comes before
the figure, and the numbers follow when figures are reordered.
`[[#control-loop|the control loop]]` keeps its own text. Captions in an
embedded entity are numbered as part of the document that embeds it.

### Caption kinds

Figures, tables and equations each have their own sequence: a document with
two figures and a table has *Figure 1*, *Figure 2* and *Table 1*. A table's
caption is placed above it, the others below. The kinds are fixed; how a
caption looks is up to the stylesheet, which can style each kind through its
`captioned-figure`, `captioned-table` or `captioned-equation` class.
