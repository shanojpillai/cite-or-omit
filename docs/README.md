# Documentation

This repository contains the product requirements and chunk-owned design documents.

## Start here

- [Product requirements](prd/PRD.md)
- [Chunk 01: Ingest, normalise & segment](chunks/01-ingest/README.md)

## Documentation model

The system has seven chunks. Each chunk is a bounded design area with the same document set:

| Document | Purpose | Owner |
|---|---|---|
| `README.md` | Scope, status, dependencies and reading order | Chunk |
| `hld.md` | System boundary, flow, contracts and major trade-offs | Chunk |
| `patterns.md` | Reusable architecture, design and reliability patterns | Chunk |
| `lld.md` | Components, data model, algorithms, APIs, tests and runbook | Chunk |
| `infrastructure.md` | Deployment topology and cloud-specific mapping | Chunk |
| `operations.md` | CI/CD, environments, alarms, SLOs and incident procedures | Chunk |

The files are deliberately separate because they answer different questions. The HLD explains what the design is; the patterns document explains why the recurring structures fit; the LLD explains how to build it; infrastructure explains where it runs; operations explains how it is released and operated.

## Product-wide material

These concerns stay outside chunk folders because they apply across the system:

- `prd/`: product goals, personas, functional requirements, quality gates and cross-chunk requirements.
- Future fixtures should remain synthetic or public source material only.

## Naming convention

Use lowercase stable filenames inside a chunk folder. Put the human-readable title in the document heading. This keeps links stable when a title changes and makes the same path work for chunks 02 through 07.