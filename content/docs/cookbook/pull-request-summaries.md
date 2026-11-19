---
title: "Pull-request summaries"
description: "Show reviewers what a pull request changes in the design history file, on the CI run's page, with every touched page redlined as an artifact."
---

A reviewer approving a pull request on a design history file needs to
know what it changes in the records, not only in the files: which
requirements changed and how, and which documents now read differently.
In this recipe every pull request's CI run shows that on its summary
page, and keeps the redlined pages as an artifact to attach to the
change record.

The [example repository](https://github.com/dynamatt/provenance-example)
implements this recipe: see `templates/diff/pr-changes.md.tmpl` and
`.github/workflows/validate.yml`.

## 1. A Markdown output

`provenance diff` writes any template in `templates/diff/` as an output
([Reviewing changes]({{< relref "docs/concepts/reviewing-changes#your-own-outputs" >}})).
Write one that renders the comparison as Markdown, here as a table of
records:

```gotemplate
{{- /* templates/diff/pr-changes.md.tmpl */ -}}
## Design history changes

From `{{short .Old.Commit}}` to `{{short .New.Commit}}`: {{summary .}}.
{{- if .Entities}}

| Entity | Type | Change | What changed |
| --- | --- | --- | --- |
{{- range .Entities}}
| **{{.ID}}** | {{.Type}} | {{if .Derived}}derived{{else}}{{.Op}}{{end}} | {{if or (eq .Op "added") (eq .Op "removed")}}{{cell .Title}}{{end}}
{{- range $i, $f := .Fields}}{{if $i}}; {{end}}{{.Label}}:
{{- if .Text}} {{plural .Text.Changed "block" "blocks"}} changed
{{- else if or .Added .Removed}}{{with .Added}} + {{join .}}{{end}}{{with .Removed}} − {{join .}}{{end}}
{{- else}} {{with .Old}}{{cell .}}{{else}}(none){{end}} → {{with .New}}{{cell .}}{{else}}(none){{end}}{{end}}
{{- end}} |{{end}}{{end}}
{{- if .Files}}

Other files:
{{range .Files}}
- `{{.Path}}` {{.Op}}{{end}}{{end}}
```

The name ends `.md`, so it is a text template: nothing is escaped.
`cell` keeps a value inside its table cell, escaping `|` and joining
lines. Try it locally against any two refs:

```bash
provenance diff main...HEAD --output pr-changes.md
```

## 2. The CI step

Add a step to the workflow that runs on pull requests. The merge base
needs the history, so check out every commit:

```yaml
      - name: Check out this repository
        uses: actions/checkout@v7
        with:
          # diff compares the pull request with its merge base.
          fetch-depth: 0

      # ... install Provenance and validate, as before ...

      - name: Diff
        if: ${{ github.event_name == 'pull_request' && !cancelled() }}
        run: |
          provenance diff "origin/$GITHUB_BASE_REF...HEAD" \
            --output pr-changes.md=pr-changes.md \
            --output html=diff.html
          cat pr-changes.md >> "$GITHUB_STEP_SUMMARY"

      - name: Upload the rendered diff
        if: ${{ github.event_name == 'pull_request' && hashFiles('diff.html') != '' }}
        uses: actions/upload-artifact@v7
        with:
          name: diff
          path: diff.html
```

The summary is appended to the run's summary page, after `validate`'s, so
both show. `diff` writes nothing unless every output renders, and exits
`0` whether or not anything changed: it informs the review, and
`validate` stays the gate.

## 3. Reviewing

On the pull request, open the run's summary page for the table of
changes, and download the `diff` artifact for the redlined pages. The
artifact names both commits and their content hashes, so it can be
attached to the change record as what was reviewed.

## Variations

- **A release diff.** `provenance diff v1.0 v1.1 --output html=release.html`
  compares two release tags. The template is read from the later tag, or
  from the working tree when the tag predates it.
- **One document.** `--scope DOC/DOC-0006.md` limits the comparison to
  the records a document shows.
- **Failing on change.** `--exit-code` exits `1` when anything changed,
  for a job that must notice an unexpected change, such as one checking
  that a released baseline is untouched.
