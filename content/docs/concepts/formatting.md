---
title: "Formatting"
description: "The one canonical form of an entity file, which provenance fmt writes, so edits to different fields merge cleanly."
---

Every entity file has one canonical form. When two people edit different
fields of a file written that way, git merges their changes on its own.
`provenance fmt` writes files in that form, and `provenance validate` warns
of a file that isn't.

## What canonical means

Only the frontmatter is formatted. The Markdown body is kept as written,
except that the file gets LF line endings and exactly one final newline.

| Rule | Canonical form |
| --- | --- |
| Key order | `id`, `type`, then the type's fields in the order the schema declares them. A key the schema doesn't declare is kept after them. |
| Layout | One field per line, two-space indents. Every list in block style, one `-` per item: links, items and list rows alike. |
| Empty fields | Left out. An omitted field reads as blank everywhere, so nothing changes but the file's length. |
| Text with line breaks | A literal block (`\|`), its lines as written. `fmt` never chooses where a line breaks. |
| Scalars | Plain, unless YAML would read a different value: then double-quoted (`"yes"`, `"#3"`). Dates as `2026-10-28`, booleans as `true` and `false`, numbers in their shortest form. |

Before:

```yaml
---
type: Requirement
id: REQ-0042
status: draft
title: 'Signed exports'
implements: [USR-0003]
deferred:
order: 42
---
```

After `provenance fmt`:

```yaml
---
id: REQ-0042
type: Requirement
title: Signed exports
status: draft
implements:
  - USR-0003
order: 42
---
```

A comment moves with its key, and `fmt` never drops one. A value that isn't
of its field's type is kept as written: `validate` reports it, and fixing it
is the author's call.

### Folded text

A folded block (`>`) is the one YAML form that reflows: its line breaks
read as spaces. `fmt` writes a folded text value as a literal block that
keeps the author's line breaks. The value gains newlines where it had
spaces, and Markdown renders a line break inside a paragraph as a space, so
the paragraph reads the same.

## Running fmt

```bash
provenance fmt
```

formats the entity files changed since the last commit: staged, unstaged
or untracked. A file nobody touched never changes by accident, which
matters because a rewrite shows in that entity's history as an edit by
whoever ran `fmt`. Name paths to narrow it further:

```bash
provenance fmt REQ/
```

`--all` formats every entity file. Use it deliberately, in a commit of its
own, such as after reordering a type's fields in its schema:

```bash
provenance fmt --all
```

Before writing a file, `fmt` reads it back and checks that every value is
unchanged and that formatting it again changes nothing. A file that fails
this check is not written. A file whose frontmatter doesn't parse, or whose
type isn't declared, is skipped and reported.

## At the merge gate

`validate` reports each file not in canonical form as a `content/format`
warning, which doesn't fail the gate. To make formatting block a merge, add
a step after `validate`:

```bash
provenance fmt --all --check
```

It writes nothing, lists each file not in canonical form, and exits 1 if
there are any. Both report the same files, so they never disagree.
