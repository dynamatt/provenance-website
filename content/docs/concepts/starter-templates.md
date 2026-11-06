---
title: "Starter templates"
description: "What provenance init writes, the regulatory clauses its fields cite, and switching on its CI gate."
---

`provenance init` starts a design history file from a starter template. The
starter is ordinary project files: from the moment they are written, they
are yours to change or delete.

```bash
provenance init my-device --code MYDEV --name "My Device"
```

`--code` and `--name` fill `.component`. The code is required: 1 to 8
upper-case letters and digits, used in component-qualified IDs. The name
defaults to the folder's name, as written. `init` writes nothing if any of its files is already in the folder,
or if the folder is already inside a Provenance repository. When the
folder isn't in a git repository, it runs `git init` on branch `main`; it
never commits, because the first commit is a record with your name on it.

## The medical-device starter

| Folder | Type | Records |
| --- | --- | --- |
| `USR/` | User Need | What a user or patient needs, in their terms |
| `REQ/` | Requirement | Design inputs, one shall-statement each |
| `DES/` | Design | Design outputs, traced to the inputs they meet |
| `RSK/` | Risk | Hazards, harms, severity and probability, and their controls |
| `VER/` | Verification Protocol, Verification Evidence | How outputs are verified, and each run's results |
| `VAL/` | Validation Evidence | How the device was shown to meet its users' needs |
| `ECO/` | Change Request | Design changes and what they affect |
| `REF/` | Reference | Standards and other external documents |
| `DOC/` | Document | Documents assembled from the records with query blocks |

Each entity in the starter is titled "Example" and shows how one record
links to the next: replace them with your own. `DOC-0001`, the design and
development plan, lists every record of each type, so it stays current as
you add them.

`rules/` holds the checks a design review makes: a requirement with no
verification, a protocol that verifies nothing, an unacceptable risk with no
risk-benefit analysis, among others. `provenance validate` runs them all.

## Clause citations

Every field's help ends with the clause it serves, so the schema shows why
each field exists:

```yaml
- name: statement
  type: text
  body: true
  required: true
  help: "One verifiable shall-statement. Inputs are complete, unambiguous and able to be verified or validated. (ISO 13485:2016 §7.3.3)"
```

Design control fields cite ISO 13485:2016, and risk fields ISO 14971:2019.
When you add a field, cite its clause the same way.

## The CI gate

A starter written by a release includes `.github/workflows/validate.yml`,
which runs `provenance validate` and `provenance fmt --all --check` on every
pull request. It downloads the same release, pinned by its hash, which you
set once: copy the `provenance-linux-amd64` line's hash from that release's
`SHA256SUMS` into `PROVENANCE_SHA256`. Until you do, the check fails and says
so. Then make it a required check on `main`.
