---
title: "Git and the audit trail"
description: "Why the repository is the source of truth for Provenance entities."
---

## Repository as source of truth

Provenance is designed so that entities live in the team's Git repository.
The repository's history captures reviewed changes, and its normal access
controls and collaboration practices remain part of the team's workflow.
The product does not require a Provenance-operated server in the critical
path for creating, reading, or verifying entities.

## Integrity evidence

The product's architecture uses cryptographic hashes to identify the
software used to work with entities and to support integrity checks. A hash
can help establish that content matches a particular state; by itself, it
does not establish who approved a change, whether a process was followed, or
that a system is compliant.

## Shared responsibility

Teams must define and validate how repository permissions, reviews, releases,
backups, and electronic records fit their own quality system. Product
capabilities and validation guidance are pre-release and may change as
development progresses.

capabilities and validation guidance are pre-release and may change as
