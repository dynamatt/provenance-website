---
title: "Stale evidence"
description: "Report evidence whose protocol changed after it ran, until it is run again or the change is justified, and protocols changed after their approval."
---

Evidence shows that a test passed as the protocol stood when it ran. When
the protocol changes afterwards, the evidence no longer shows that, until
someone runs the test again or records why the change doesn't matter.
This recipe makes that a rule at the merge gate, from the repository's own
history, with no version numbers to keep:

- A protocol records its approvals, each with a date.
- Evidence records justifications, each with a date and a reason.
- Four rules report:
  - evidence run, or last justified, before its protocol changed;
  - a protocol changed after its latest approval;
  - evidence run before its protocol's latest approval;
  - an execution date after the commit that recorded it.

The [example repository](https://github.com/dynamatt/provenance-example)
implements this recipe: see `schema/VerificationProtocol.yaml`,
`schema/Evidence.yaml`, `rules/` and `VER/`. EVD-0001 is justified, and
EVD-0002 is left stale on purpose.

## The schema

The protocol's approvals, and the latest of them:

```yaml
# schema/VerificationProtocol.yaml (extract)
- name: approvals
  type: list
  fields:
    - {name: approver, type: string, required: true}
    - {name: date, type: date, required: true}
    - {name: meaning, type: string, required: true}
- name: approved_on
  type: calculated
  formula: "MAX(approvals[].date)"
```

The evidence's justifications, and the day it last held, run or
justified:

```yaml
# schema/Evidence.yaml (extract)
- name: justifications
  type: list
  fields:
    - {name: date, type: date, required: true}
    - {name: reason, type: text, required: true}
- name: current_as_of
  type: calculated
  formula: "MAX(execution_date, justifications[].date)"
```

An approval here is the record's own data, a row a person committed. It is
not an electronic signature. When signatures come, a rule names
`signatures` where it named `approvals`.

## The rules

Stale evidence: the protocol's procedure or steps changed after the
evidence was run or justified. `kind: changed` leaves out the commit that
added the protocol, so records imported into git after they were run
aren't stale on import day.

```yaml
# rules/evidence-current.yaml
severity: warning
message: "the protocol it executed has changed since it was run or justified"
from: VerificationEvidence
where:
  field: tests
  where:
    field: changes
    where:
      - {field: kind, operator: equals, value: changed}
      - any_of:
          - {field: fields, operator: equals, value: procedure}
          - {field: fields, operator: equals, value: steps}
      - {field: date, operator: greater_than, value_of: ^^current_as_of}
```

A protocol changed after its latest approval. Changing its `status` or its
approvals is no change to what it tests:

```yaml
# rules/approval-current.yaml
severity: error
message: "the procedure or steps changed after the latest approval"
from: VerificationProtocol
where:
  field: changes
  where:
    - {field: kind, operator: equals, value: changed}
    - any_of:
        - {field: fields, operator: equals, value: procedure}
        - {field: fields, operator: equals, value: steps}
    - {field: date, operator: greater_than, value_of: ^approved_on}
```

Evidence run, and not justified since, before its protocol's latest
approval:

```yaml
# rules/evidence-after-approval.yaml
severity: warning
message: "run or justified before the protocol's latest approval"
from: VerificationEvidence
where:
  field: tests
  where: {field: approved_on, operator: greater_than, value_of: ^current_as_of}
```

An execution date after the commit that recorded it. The rule counts the
changes that set the date on or after it. None means the change that wrote
it came first. Testing each change instead would report evidence run again
later.

```yaml
# rules/execution-not-in-future.yaml
severity: error
message: "execution date is after the commit that recorded it"
from: Evidence
where:
  - {field: execution_date, operator: exists}
  - field: changes
    where:
      - {field: fields, operator: equals, value: execution_date}
      - {field: date, operator: greater_or_equal, value_of: ^execution_date}
    aggregation: {type: count}
    operator: equals
    value: 0
```

## Clearing a report

| To clear | Do |
| --- | --- |
| `evidence-current` | Run the test again and record the new `execution_date`, or add a justification dated on or after the protocol's change |
| `approval-current` | Add an approval dated on or after the change |
| `evidence-after-approval` | Run the test again after the approval, or justify the evidence after it |

## In CI

These rules read the dates of the commits that changed each entity, so
Validate needs every commit: check out with `fetch-depth: 0`. In a shallow
clone, `validate` stops with exit 2 and says so.

Merge pull requests with a merge commit or rebase them. Either way, a
change keeps the date it was written, so a justification written in the
same pull request comes after it. A squash merge dates everything it
holds the day it was merged, and a justification written before that day
then comes before its own change.
