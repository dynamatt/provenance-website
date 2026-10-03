---
title: "Calculated fields"
description: "Fields computed from a formula, such as a risk rating, never stored or hand-edited."
---

A calculated field's value is computed from a formula every time it is
read. It is never written in the entity file, so it cannot be edited by
hand or fall out of date. A typical use is a risk rating computed from
linked severity and occurrence scores.

```yaml
# schema/Risk.yaml
  - name: failure_modes
    type: list
    fields:
      - name: severity
        type: link
        target: [SeverityLevel]
        cardinality: one
      - name: occurrence
        type: link
        target: [OccurrenceLevel]
        cardinality: one
      - name: row_rating
        type: calculated
        formula: "IF(ISBLANK(occurrence.score), severity.score * 5, severity.score * occurrence.score)"
  - name: overall_risk_rating
    type: calculated
    formula: "MAX(failure_modes[].row_rating)"
```

`row_rating` is calculated on each failure mode, and `overall_risk_rating`
takes the largest of them. Change a linked occurrence level's `score` and
both update on the next export.

## Formulas

Formulas are written as in a spreadsheet:

| Element | Syntax |
| --- | --- |
| Arithmetic | `+ - * /` and parentheses |
| Comparison | `= <> < > <= >=`, giving `TRUE` or `FALSE` |
| Numbers, text, truth values | `2.5`, `"high"` (write `""` for a quote inside text), `TRUE`, `FALSE` |
| Logic | `IF(condition, then, else)`, `AND(…)`, `OR(…)`, `NOT(x)` |
| Blank test | `ISBLANK(x)` |
| Aggregates | `MAX(…)`, `MIN(…)`, `SUM(…)`, `COUNT(…)`, `AVG(…)` |

Function names, `TRUE` and `FALSE` can be written in any case, and a
leading `=` is allowed.

## Referring to fields

| Formula | Reads |
| --- | --- |
| `order` | A field of the same entity. In a list row's formula, a field of the same row. |
| `severity.score` | A field of the linked entity, through a link with `cardinality: one`. Links can be chained. |
| `failure_modes[].row_rating` | That field of every row of a list. |
| `failure_modes[].severity.score` | A field of the entity each row links to. |
| `mitigated_by.order` | A field of every entity linked through a link with `cardinality: many`. |
| `verified_by` | The entities linking to this one through a reverse link. |
| `readings` | Every value of a list that holds plain values. |

A calculated field can read other calculated fields, including on linked
entities and list rows.

A reference that can have several values, such as `failure_modes[].row_rating`,
`mitigated_by.order` or `verified_by`, can only be used inside an aggregate
or `ISBLANK`.

## Blank values

A field with no value is blank. Blank behaves differently from a
spreadsheet, which treats a blank cell as `0`:

- Arithmetic or comparison with a blank is blank, so a missing occurrence
  score never turns a risk rating into `0`. Test for it with `ISBLANK`, as
  in the example above.
- So are arithmetic on text and division by zero.
- `IF` needs a `TRUE` or `FALSE` condition. `AND`, `OR` and `NOT` need
  `TRUE` or `FALSE` arguments; a blank argument makes them blank unless
  another argument decides the result, as in `AND(FALSE, x)`.

## Aggregates

- `MAX`, `MIN`, `SUM` and `AVG` use the numbers among their arguments and
  skip everything else.
- `COUNT` counts every value that is not blank, numbers or not, so
  `COUNT(verified_by)` is the number of verification protocols.
- With no values, `SUM` and `COUNT` are `0`; `MAX`, `MIN` and `AVG` are
  blank.
- Every row, list item and linked entity counts once, so two failure modes
  both rated 15 add up to 30.

Arguments can be mixed: `MAX(failure_modes[].row_rating, minimum_rating)`.

## Using calculated values

A calculated field is shown on entity pages, can be used in a
[query block's]({{< relref "docs/concepts/documents" >}}) `where`,
`order_by` and `render: field:<name>`, and in `[[ID#field]]`, exactly like a
stored field. In a template it is a number, `true`/`false` or text, and
empty when blank.

## Problems

A formula that cannot be evaluated does not stop an export. The entity page
shows the formula with the problem, and other formulas reading the field
see it as blank:

- a syntax error: `formula: column 8: unexpected end of formula`
- an unknown field or function
- a reference with several values used outside an aggregate
- a formula that depends on itself, directly or through other calculated
  fields: `formula depends on itself: Risk.a → Risk.b → Risk.a`

These will also be reported by `provenance validate` once it is
implemented.
