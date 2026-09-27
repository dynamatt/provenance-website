# Provenance Website

The Provenance website — Hugo with hand-built marketing pages and a
documentation section powered by Hugo Book. Marketing layout markup lives
in `layouts/`; documentation prose lives in `content/docs/`.

## Run locally

Requires the **extended** Hugo binary (the site uses no Sass, but the
extended build is what the CI workflow installs, so match it locally):

```sh
hugo server
```

Initialize the Hugo Book submodule on a fresh clone, then start the site:

```sh
git submodule update --init --recursive
hugo server
```

Then open <http://localhost:1313/>. Documentation is available at
<http://localhost:1313/docs/>.

## Lint

```sh
npm install
npm run lint
```

Lints Markdown (`content/*.md`, `README.md`) with markdownlint-cli2; rules
live in `.markdownlint-cli2.jsonc`. `.github/workflows/lint.yml` also runs
actionlint against the workflow files. Both run in CI on every push and
pull request.

## Deploy

`.github/workflows/gh-pages.yml` builds and deploys to GitHub Pages on
every push to `main`. One-time setup in the repo: **Settings → Pages →
Source → GitHub Actions**. The workflow sets `--baseURL` automatically
from `actions/configure-pages`, so it works whether this ends up on a
project page (`https://<org>.github.io/provenance-site/`) or a custom
domain — no config changes needed either way. `hugo.toml`'s `baseURL` is
only the fallback for local runs.

## Known placeholders to revisit

- The "Docs" link in the nav (`layouts/partials/nav.html`) points at
  `github.com/dynamatt/provenance/tree/main/docs` as a stand-in until
  there's a real documentation site or section.
- `content/*.md` front matter and copy throughout assumes the CLI repo
  is `github.com/dynamatt/provenance` — update if that changes.
- Hugo version pinned in the workflow (`HUGO_VERSION`) is a recent
  release as of when this was scaffolded; bump it periodically.

## Adding a marketing page

1. Add `content/<slug>.md` with front matter: `title`, `description`,
   `layout: "<slug>"`, `nav: "<slug>"`.
2. Add `layouts/_default/<slug>.html` with `{{ define "main" }} ... {{ end }}`.
3. Add a nav link for it in `layouts/partials/nav.html` (and the footer's
   site-links column, if it should appear there too).

## Adding documentation

Add Markdown pages under `content/docs/`. Use `_index.md` files to create
sections and regular `.md` files for pages; Hugo Book builds the docs
navigation from this content tree.

## Design tokens

Colors, fonts, and shared component styles (nav, buttons, cards, table,
the responsive grid/row classes) live in one place:
`layouts/partials/head.html`. Page templates use those classes and CSS
variables rather than repeating raw hex values, so a palette or type
change is a one-file edit.
