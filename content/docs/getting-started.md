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

A binary whose hash doesn't match is not the one that was published: don't
run it.

Each release has a binary for three platforms, and each one is run on its
own platform before the release is published:

| Platform | File |
| --- | --- |
| Linux, x86-64 | `provenance-linux-amd64` |
| macOS, Apple silicon | `provenance-darwin-arm64` |
| Windows, x86-64 | `provenance-windows-amd64.exe` |

The binaries aren't yet signed by Apple or Microsoft. The SHA-256 check is
what proves a download is the published binary, so check it first, then:

- **macOS:** check with `shasum -a 256 provenance-darwin-arm64` and compare
  the result with that file's line in `SHA256SUMS`. Gatekeeper blocks a
  downloaded binary that isn't signed, so clear its quarantine before the
  first run:

  ```bash
  xattr -d com.apple.quarantine provenance-darwin-arm64
  chmod +x provenance-darwin-arm64
  ./provenance-darwin-arm64 version
  ```

- **Windows:** check with
  `Get-FileHash provenance-windows-amd64.exe -Algorithm SHA256` in
  PowerShell and compare it with that file's line in `SHA256SUMS` (case
  doesn't matter). SmartScreen may warn that the file is unrecognized:
  choose *More info*, then *Run anyway*.

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
