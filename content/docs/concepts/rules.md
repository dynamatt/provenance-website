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

## Worked examples

Each of these rules is in the example repository's
[`rules/`](https://github.com/dynamatt/provenance-example/tree/main/rules)
folder, one for each kind of check a design control process usually needs.
The rule at the top of this page is the first kind: a field required only
under a condition. A link to an entity that doesn't exist, or to one of a
type its field doesn't allow, needs no rule: the built-in `links/…` checks
report it.

### Values in a set

`not_in` reports a value outside a list, here of a plain text field that
isn't an enum. It also matches an entity with no value:

```yaml
# rules/eco-change-type-valid.yaml
severity: error
message: "change_type must be one of the approved categories"
from: ChangeRequest
where:
  field: change_type
  operator: not_in
  value: [correction, enhancement, safety]
```

### Counting links

`aggregation` counts the entities a link reaches, here the protocols that
verify a requirement, through the reverse link no requirement file writes:

```yaml
# rules/requirement-verified.yaml
severity: warning
message: "requirement missing verification"
from: Requirement
where:
  field: verified_by
  where:
    field: type
    operator: equals
    value: VerificationProtocol
  aggregation: {type: count}
  operator: less_than
  value: 1
```

### Comparing across a link

A `where` on a link tests the linked entity, and `^` reads the entity the
rule is checking. Evidence executed before the build it tested existed:

```yaml
# rules/evidence-after-build.yaml
severity: error
message: "executed before its build was built"
from: VerificationEvidence
where:
  field: build_tested
  where:
    field: build_date
    operator: greater_than
    value_of: ^execution_date
```

### Comparing within an entity, row by row

A `where` on a list tests each row on its own and reports each one that
fails at its line. Equipment whose calibration had lapsed when the test ran:

```yaml
# rules/equipment-calibration-current.yaml
severity: error
message: "equipment calibration was not current at time of test execution"
from: VerificationEvidence
where:
  field: equipment_used
  where:
    field: calibration_due_date
    operator: less_than
    value_of: ^execution_date
```

### No cycles

`transitive: true` follows a link to any depth. A requirement that is its
own ancestor, reported on every requirement of the cycle:

```yaml
# rules/no-requirement-decomposition-cycles.yaml
severity: error
message: "requirement decomposition hierarchy contains a cycle"
from: Requirement
where:
  field: parent_requirement
  transitive: true
  where:
    field: id
    operator: equals
    value_of: ^id
```

### Uniqueness

A `where` with `from` looks among other entities, which no link reaches.
Requirements sharing a title, each reported with the others it shares it
with:

```yaml
# rules/requirement-title-unique.yaml
severity: warning
message: "two requirements share the same title — likely a duplicate or copy-paste error"
from: Requirement
where:
  from: Requirement
  where:
    - field: title
      operator: equals
      value_of: ^title
    - field: id
      operator: not_equals
      value_of: ^id
```

### Anything a query can find

Any query is a rule: a project states its own conventions the same way.
Requirements that implement a deprecated user need:

```yaml
# rules/no-implements-deprecated-need.yaml
severity: warning
message: "requirement implements a deprecated user need"
from: Requirement
where:
  field: implements
  where:
    field: status
    operator: equals
    value: deprecated
```

### Code block languages

Every entity has a `code_blocks` list, one row per fenced block, with its
`language` (`none` for a fence with none). This limits the languages the
project's Markdown may use:

```yaml
# rules/block-languages.yaml
severity: error
message: "code block in a language this project does not use"
from: any
where:
  field: code_blocks
  where:
    field: language
    operator: not_in
    value: [query, caption, mermaid, text, none]
```

A diagram in a language this version can't render, such as `mermaid`, is
reported by the built-in `content/unrendered-block` check whatever the
rules allow, so it is never published as source text.

## Reports for tools

`--output <name>` writes the same violations for other tools. The exit
code is the same whichever you choose, and only the exit code is
guaranteed:

| Format | For |
| --- | --- |
| `text` | People. The default: one line per violation, as above. |
| `json` | Scripts: `version`, `violations` (each with `rule`, `severity`, `message`, `entity`, `path`, `line`) and `summary` (`errors`, `warnings`, `rules_skipped`). |
| `junit` | CI test reports. Every built-in check and every rule is a test case. An error fails its case; warnings are in its output, so they don't fail it. A rule that wasn't run is skipped. |
| `sarif` | Code scanning: SARIF 2.1.0, with every check and rule described and each violation at its file and line. |

`--output <name>=<file>` writes the report to a file instead, creating its
folders. Repeat `--output` to write several reports from one run: at most
one goes to standard output, and with no `--output` at all the text report
does.

On GitHub, upload the SARIF report so pull requests show each violation on
its line, while the text report goes to the job's log. The job needs
`permissions: security-events: write`. The upload runs even when the gate
fails, as long as a report was written:

```yaml
- run: provenance validate --output text --output sarif=validate.sarif
- uses: github/codeql-action/upload-sarif@v4
  if: ${{ !cancelled() && hashFiles('validate.sarif') != '' }}
  with:
    sarif_file: validate.sarif
```

On GitLab, a JUnit report lists the checks in the pipeline's *Tests* tab:

```yaml
validate:
  script: provenance validate --output text --output junit=validate.xml
  artifacts:
    when: always
    reports:
      junit: validate.xml
```

Each run gives the same bytes for the same commit: no timestamps, and
every path relative to the repository root.

## Custom reports

A project can write its own report as a template in `templates/reports/`.
`--output <name>` with any name other than a format renders
`templates/reports/<name>.tmpl`, so no report can be called `text`, `json`,
`junit` or `sarif`. A name ending `.html` is an
[`html/template`](https://pkg.go.dev/html/template), which escapes what it
shows for HTML. Any other, such as `pr-summary.md`, is a
[`text/template`](https://pkg.go.dev/text/template), which escapes nothing.
The exit code is the same whichever reports you write.

A report template receives:

| Name | Holds |
| --- | --- |
| `.Violations` | Every violation, in the order the text report lists them, each with `.Rule`, `.Severity`, `.Message`, `.Path`, `.Line` and `.Entity`. |
| `.Rules` | Every built-in check, then every rule file, so a report can list the ones that passed. Each has `.ID`, `.Severity`, `.Message`, `.Builtin`, `.Path` (the rule file), `.Query` (its text), `.Skipped` (why it wasn't run, or empty) and its own `.Violations`. |
| `.Summary` | `.Errors`, `.Warnings` and `.RulesSkipped`. |
| `.Component` | `.Code` and `.Name`. |
| `.GitSHA`, `.ContentHash`, `.Dirty` | The commit, the content hash (ending `-dirty` when the working tree differs from it) and whether it differs. Empty outside a git repository. |

`.Entity` is the entity the violation is about, with the same fields as in
a [type template]({{< relref "docs/concepts/templates#type-templates" >}}),
or nothing when the violation is about no entity, such as a problem in a
rule file. A built-in check's `.Message` is its short description.

Reports have the template functions `link`, `href` and `short`, but not
`markdown`. Links are relative to the exported site's main page, so a report
written there links to each entity's page:

```sh
provenance export website
provenance validate --output validation.html=_site/validation.html
```

Export first: it empties its folder, and `--output` never does. A report
listing every check, passed or not:

```html
<table>
{{- range .Rules}}
<tr>
  <td><code>{{.ID}}</code> {{.Message}}</td>
  <td>{{if .Skipped}}not run{{else if .Violations}}{{len .Violations}} violations{{else}}passed{{end}}</td>
</tr>
{{- end}}
</table>
<ul>
{{- range .Violations}}
  <li>{{.Severity}}: {{link .Entity}} {{.Message}} (<code>{{.Path}}:{{.Line}}</code>)</li>
{{- end}}
</ul>
```

A merge gate can write every report it needs from one run. On GitHub, a
Markdown report written to `$GITHUB_STEP_SUMMARY` shows on the run's
summary page:

```yaml
- run: >-
    provenance validate --output text
    --output sarif=validate.sarif
    --output pr-summary.md="$GITHUB_STEP_SUMMARY"
```

The example repository's
[*Validate* workflow](https://github.com/dynamatt/provenance-example/blob/main/.github/workflows/validate.yml)
is a complete gate built this way.

Every run of `validate` renders every report template, whether or not it
writes it, so a broken one is caught at the merge gate. One that can't be
parsed, or names something the data doesn't have, such as `.Summary.Failures`,
is a `content/template` error at its line. Every report is rendered before
any is written, so if one fails the run writes nothing. A name that is
neither a format nor a report template is a usage error, exit code `2`.

## Requiring the check

`validate` is a merge gate only when the git host won't merge a change
that fails it. Run it from a released binary, pinned to its version and
the SHA-256 that version's `SHA256SUMS` gives it, so the gate is the binary
you checked. The binary is the whole gate: the workflow only downloads it,
checks it and runs it.

On GitHub:

```yaml
# .github/workflows/validate.yml
name: Validate
on:
  pull_request:
  push:
    branches: ["main"]
permissions:
  contents: read
jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    env:
      PROVENANCE_VERSION: v0.2.0-alpha
      # provenance-linux-amd64's line in that release's SHA256SUMS
      PROVENANCE_SHA256: <sha256>
    steps:
      - uses: actions/checkout@v7
      - name: Install Provenance
        run: |
          curl -fsSLo provenance "https://github.com/dynamatt/provenance/releases/download/$PROVENANCE_VERSION/provenance-linux-amd64"
          echo "$PROVENANCE_SHA256  provenance" | sha256sum -c -
          install -D -m 0755 provenance "$RUNNER_TEMP/bin/provenance" && rm provenance
          echo "$RUNNER_TEMP/bin" >> "$GITHUB_PATH"
      - name: Validate
        run: >-
          provenance validate --output text
          --output sarif=validate.sarif
          --output pr-summary.md="$GITHUB_STEP_SUMMARY"
      - uses: github/codeql-action/upload-sarif@v4
        if: ${{ !cancelled() && hashFiles('validate.sarif') != '' }}
        with:
          sarif_file: validate.sarif
```

Then, in the repository's *Settings › Rules › Rulesets* (or *Branches*),
protect the main branch and require the `validate` status check to pass
before merging. The example repository's
[*Validate* workflow](https://github.com/dynamatt/provenance-example/blob/main/.github/workflows/validate.yml)
is this gate.

On GitLab, run the same commands in a job, and turn on *Settings › Merge
requests › Pipelines must succeed*:

```yaml
validate:
  image: alpine:3
  before_script:
    - apk add --no-cache curl
  variables:
    PROVENANCE_VERSION: v0.2.0-alpha
    PROVENANCE_SHA256: <sha256>
  script:
    - curl -fsSLo provenance "https://github.com/dynamatt/provenance/releases/download/$PROVENANCE_VERSION/provenance-linux-amd64"
    - echo "$PROVENANCE_SHA256  provenance" | sha256sum -c -
    - chmod +x provenance
    - ./provenance validate --output text --output junit=validate.xml
  artifacts:
    when: always
    reports:
      junit: validate.xml
```

## Stricter review for schema and rules

A change to `schema/` or `rules/` changes what every entity is checked
against, so it deserves stricter review than a change to content. That
review is a git host setting, not something `validate` can enforce. On
GitHub and GitLab, name the reviewers in a `CODEOWNERS` file:

```text
# .github/CODEOWNERS (GitHub) or CODEOWNERS (GitLab)
/schema/  @your-org/quality
/rules/   @your-org/quality
```

Then require their approval: on GitHub, *Require review from Code Owners*
in the main branch's ruleset or protection rule; on GitLab, *Code owner
approval* on the protected branch. A pull request that changes a rule then
can't merge until one of them approves it.
