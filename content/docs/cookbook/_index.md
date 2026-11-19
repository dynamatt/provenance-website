---
title: "Cookbook"
description: "Recipes for patterns teams commonly need, built from Provenance's schema, links, query blocks and templates."
---

Each recipe solves one common need with the parts Provenance already has:
a schema type, links, query blocks and templates. A recipe is a pattern,
not a built-in feature. Copy it into your repository and adapt it. Each
one is implemented in the
[example repository](https://github.com/dynamatt/provenance-example), so
you can export it and see the result.

- [Glossary]({{< relref "docs/cookbook/glossary" >}}): terms defined once as
  entities, cited inline in any document, and listed in a *Definitions*
  section at the end of each document that uses them.
- [Reusable documents]({{< relref "docs/cookbook/reusable-documents" >}}):
  a kind of document written once, as a type extending Document and a
  template filling a block of Document's, so each report holds only its
  prose and what it covers.
- [Pull-request summaries]({{< relref "docs/cookbook/pull-request-summaries" >}}):
  what a pull request changes in the design history file, as a table on
  the CI run's page, and every page it touches redlined as an artifact.
