---
title: "Reviewing changes"
description: "See what a change did to the design history file, record by record and as the reader of the exported site sees it, between any two git refs."
---

`git diff` shows what a change did to files. `provenance diff` shows what
it did to the design history file: which records were added, removed or
changed, which fields changed and how, and which pages of the exported
site now read differently. It compares any two git refs, so the same
command reviews a pull request, a working tree before committing, or two
releases.

```bash
provenance diff main...HEAD
```

```text
diff merge base of main and HEAD (5a845df) → HEAD (9c41e07)
REQ-0002  Requirement  changed
  statement:
    ~ … exceeding [-8-] {+6+} mA under …
  status: approved → in_review
1 entity: 1 changed
```

## Choosing what to compare

The refs are written as git's:

| Command | Compares |
| --- | --- |
| `provenance diff` | `HEAD` with the working tree: what you haven't committed |
| `provenance diff main` | `main` with the working tree |
| `provenance diff v1.0 v1.1` | Two refs, here two release tags |
| `provenance diff v1.0..v1.1` | The same |
| `provenance diff main...HEAD` | `HEAD` with its merge base with `main`: what a pull request from `HEAD` into `main` changes, not what has landed on `main` since |

A ref is anything git resolves: a branch, a tag, a commit, `HEAD~2`,
`origin/main`. Each side is read from git's objects, with its own schema,
rules and templates; nothing is checked out, and the working tree is left
alone.

`--scope` limits the comparison to the entities an entity file or query
file selects, as it limits an
[export]({{< relref "docs/concepts/exporting" >}}). It is resolved at each
side, and an entity in either side's scope is compared.

## What counts as a change

Records are matched by ID, whatever file holds them:

| Change | When |
| --- | --- |
| added | Only the later side has the ID |
| removed | Only the earlier side has it |
| retyped | Both have it, with different types |
| changed | Both have it, and a value differs |

Values are compared as the schema types them, so a change of formatting
alone is no change: reordering a file's keys, quoting a value, rewrapping
a paragraph, or moving the file to another folder. A reformatting commit
made by [`provenance fmt`]({{< relref "docs/concepts/formatting" >}})
shows as `no changes`.

For a changed record, each field that differs is shown:

- a single value, link or enum as `old → new`;
- a many-link or list as the items added and removed;
- a list of records row by row;
- the body and text fields block by block (a paragraph, a list item, a
  table row, a heading), and a changed block word by word.

[Calculated fields]({{< relref "docs/concepts/calculated-fields" >}})
and the results of
[query fields]({{< relref "docs/concepts/schema#query-fields" >}}) are
compared too, and marked *derived*: a report whose scope grew because a
requirement was approved shows as changed, though nobody edited it.

Files that aren't records, such as the schema, rules, templates and
images, are listed by path. Images are compared by their git object hash.

## The rendered diff

`--output html` writes one self-contained HTML file: the change list, then
every page of the exported site that the change touched, rendered at each
side through that side's own templates and marked:

- words removed struck through, words added highlighted;
- a removed paragraph, row or list item struck through where it was;
- unchanged parts folded, with a button to show the whole page.

```bash
provenance diff main...HEAD --output text --output html=diff.html
```

This shows changes `git diff` can't: a document whose traceability table
gained a row because a protocol was added, or a list of open items that
now includes a requirement moved back to review. Those pages are marked
*page only*: their own files didn't change, but what they show did.

The file names both commits and their
[content hashes]({{< relref "docs/concepts/exporting" >}}) and the
version of Provenance that compared them, holds its images and styles,
and points nowhere outside itself but external links, so it can be
attached to a change record or a design review and read years later.

## Your own outputs

Each `--output` is a template over the same comparison. Besides `text` and
`html`, a project can add its own in `templates/diff/<name>.tmpl`, such as
a Markdown summary for a pull request; one named `text.tmpl` or
`html.tmpl` replaces the built-in. A template is read from the later side,
or from the working tree when the later side has none, so a template
added today also renders a diff of two old releases. The names a template
can use are listed in the design (DES-0057); the
[pull-request summary recipe]({{< relref "docs/cookbook/pull-request-summaries" >}})
is a worked example.

## Exit codes

| Code | When |
| --- | --- |
| `0` | The comparison was made, whether or not anything changed |
| `1` | With `--exit-code`, something changed |
| `2` | A ref doesn't resolve, an output doesn't exist or doesn't render, or any other error |

A side that has problems loading, such as a broken schema in an old
commit, is compared as far as it loads, and its problems are listed
first: `diff` reports, [`validate`]({{< relref "docs/concepts/rules" >}})
judges.

The [CLI reference]({{< relref "docs/cli/diff" >}}) lists the flags.
