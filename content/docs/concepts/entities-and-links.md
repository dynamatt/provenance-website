---
title: "Entities and links"
description: "How versioned entities and typed relationships form a traceability graph."
aliases: ["/docs/concepts/records-and-links/"]
---

## Entities

Requirements, design descriptions, risks, and verification evidence are
represented as entities in the project repository. Entities are plain-text
files with structured metadata, so their content can be reviewed and changed
through the same version-control workflow as other project artifacts.

## Schemas

Schemas define the fields and entity types a project uses. The product's
design goal is to keep those definitions in the repository rather than
hard-code a fixed set of entity types into the service. This lets a team
evolve its model through reviewed repository changes.

## Typed links

Links express relationships between entities, such as a requirement being
addressed by a design entity or verified by test evidence. Following these
links forms a traceability graph across the project's design-control
artifacts.

[Writing a schema]({{< relref "docs/concepts/schema" >}}) describes the schema
files, their field types, and how links and lists are declared. The
validation rules are still under development.
