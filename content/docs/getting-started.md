---
title: "Getting started"
description: "Current availability, downloading the alpha release, and how to follow Provenance development."
---

## Current availability

Provenance is pre-release. The alpha releases are for trying the product
out, not for use in a regulated process: there is no validated release or
supported production setup yet.

The product is being built in the open in the
[Provenance repository](https://github.com/dynamatt/provenance). The
website's [Get started]({{< relref "/get-started" >}}) page summarizes the
project stage.

## Trying the alpha

The `v0.1.0-alpha` release exports a design history file as a static
website. Download the binary for your platform and `SHA256SUMS` from the
[releases page](https://github.com/dynamatt/provenance/releases), and check
the binary before running it:

```bash
sha256sum -c SHA256SUMS --ignore-missing
chmod +x provenance-linux-amd64
./provenance-linux-amd64 version
```

On macOS, compare `shasum -a 256 provenance-darwin-arm64` with that file's
line in `SHA256SUMS`. A binary whose hash doesn't match is not the one that
was published: don't run it.

To see what it does, export the worked example,
[provenance-example](https://github.com/dynamatt/provenance-example):

```bash
git clone https://github.com/dynamatt/provenance-example
cd provenance-example
provenance export website
```

Open `_site/index.html`. [Exporting the DHF]({{< relref "docs/concepts/exporting" >}})
and [Templates]({{< relref "docs/concepts/templates" >}}) describe what the
export contains and how to change it.

## Before adopting the product

Treat the current documentation as product orientation only. Teams remain
responsible for their quality system, risk management, design-control
procedures, and validation of any tools used in a regulated process. Follow
the product repository for release notes and supported setup guidance.
