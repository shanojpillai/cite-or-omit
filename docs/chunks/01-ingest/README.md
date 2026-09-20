# Chunk 01 · Ingest, normalise & segment

> Turn customer files into a complete, canonical and addressable document set without silently losing pages or citation coordinates.

| | |
|---|---|
| **Chunk** | 01 of 07 |
| **PRD requirements** | [FR-ING-01 to FR-ING-05](../../prd/PRD.md#101-ingest-normalisation--segmentation) |
| **Upstream** | Consultant upload / review workspace |
| **Downstream** | Chunk 02: relevance screen & clause routing |
| **Status** | HLD complete · patterns complete · LLD complete · infrastructure complete |

## Reading order

1. [High-level design](hld.md): scope, canonical contract, flow, invariants and trade-offs.
2. [Patterns](patterns.md): the architecture, design and reliability patterns behind the HLD.
3. [Low-level design](lld.md): stages, data model, APIs, reliability, security and core test contract.
4. [Infrastructure](infrastructure.md): AWS topology, workflow, storage, networking, IAM and deployment boundaries.
5. [Test and evaluation](test-and-evaluation.md): test pyramid, ingest-quality metrics, labelled data and release gates.
6. [CI/CD](ci-cd.md): build, promotion, deployment safety, rollback and environment gates.

## Boundary

This chunk owns intake, safety checks, normalisation, rendering, text extraction, layout, segmentation, anchors and fidelity invariants. It does not decide relevance, interpret entitlement rules, extract business values, run the review workflow or own platform-wide infrastructure.

## Contracts exposed

- Canonical `Document -> Page -> Segment -> Anchor` data model.
- Versioned `document.ready/v1` and failure/status events.
- Read access to pages, segments and anchors through the ingest API.
- Fidelity guarantees: page conservation, canonical text coverage and anchor round-trip validation.

## Document map

| Concern | Canonical location |
|---|---|
| Product requirements | [PRD](../../prd/PRD.md) |
| Architecture | [HLD](hld.md) |
| Patterns and rationale | [Patterns](patterns.md) |
| Build specification | [LLD](lld.md) |
| AWS deployment | [Infrastructure](infrastructure.md) |
| Testing and evaluation | [Test and evaluation](test-and-evaluation.md) |
| CI/CD and release operations | [CI/CD](ci-cd.md) |