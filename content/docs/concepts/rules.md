---
title: "Rules"
description: "Checks a project writes as queries, which provenance validate runs at the merge gate."
---

A rule is a query: every entity it matches is a violation. Rules live in
`rules/`, one per file, and `provenance validate` runs them all, beside
its built-in checks, on every pull request. A failing rule with
`severity: error` fails the gate.

```yaml
# rules/risk-benefit-required.yaml
severity: error
message: "risk-benefit analysis required when residual acceptability is unacceptable"
from: Risk
where:
  - field: residual_acceptability
    operator: equals
    value: unacceptable
  - field: risk_benefit_analysis
    operator: missing
```

A rule file has four keys:

| Key | Meaning |
| --- | --- |
| `severity` | `error` fails the gate; `warning` is reported without failing it. |
| `message` | The text each violation is reported with. |
| `from` | The entity types the rule applies to: a type, a list of types, or `any`. |
| `where` | The condition an entity breaks the rule by, in the [condition grammar]({{< relref "docs/concepts/documents#conditions" >}}). Without it, every entity of the `from` types is a violation. |

The file name is the rule's ID: `rules/risk-benefit-required.yaml` is the
rule `risk-benefit-required`. Use lower-case words joined by hyphens. A
rule file has no `id:` key.

Because a rule is a query, the same `from` and `where` in a
[query block]({{< relref "docs/concepts/documents#query-blocks" >}})
list exactly what the rule reports. A design review checklist can embed
each rule's query, and an empty section is a passed check.

## Where a violation is reported

Each violation is reported at a file and line, so a git host can show it
on the line that breaks the rule:

```text
RSK/RSK-0001.md:2: error [risk-benefit-required] risk-benefit analysis required when residual acceptability is unacceptable (RSK-0001)
```

The line depends on how the entity matched:

| Matched by | Reported at |
| --- | --- |
| A test on a field the entity writes | The field. |
| `missing`, or a field the entity doesn't write (unset, a reverse link, a calculated field) | The entity's `id:`. |
| A `where` on a list | Each row that matched, once per row. |
| A `where` on a link | Each link value that matched, where it is written. For a reverse link, that is in the linking entity's file. |
| `transitive: true` | Each entity's own link: every requirement on a cycle is reported at its `parent_requirement`. |
| `aggregation` | The aggregated field, or the `id:` when the entity doesn't write it. |
| A `where` with `from` | The entity's `id:`, with the entities found after the message. |
| `any_of` | The first alternative that holds, by the same rules. |

When the `where` has several conditions, one of them decides: the last
`where` on a list, link or `from`, so each row or link is reported;
otherwise the last condition. Write the conditions that select what the
rule applies to first, and the check itself last. In the example above,
`risk_benefit_analysis` is `missing`, so the violation is at RSK-0001's
`id:`.

## Problems in a rule file

A rule file that can't be run is reported at its line, and only that
rule is skipped; the others still run:

| ID | Problem |
| --- | --- |
| `rules/file` | The file isn't valid YAML, or isn't a mapping. |
| `rules/name` | The file name isn't lower-case words joined by hyphens, ending `.yaml`. |
| `rules/unknown-key` | A key other than the four above, `id:` included. |
| `rules/severity` | No `severity`, or one other than `error` or `warning`. |
| `rules/message` | No `message`, or an empty one. |
| `rules/query` | No `from`, or a `from` or `where` naming a type or field the schema doesn't have. |

A schema with errors stops every rule, since rules are checked against
it, and the report says how many weren't run.
