# Chunk 01 · Ingest, normalise & segment

> Turn customer files into a complete, canonical and addressable document set without silently losing pages or citation coordinates.

| | |
|---|---|
| **Chunk** | 01 of 07 |
| **PRD requirements** | [FR-ING-01 to FR-ING-05](../../prd/PRD.md#101-ingest-normalisation--segmentation) |
| **Upstream** | Consultant upload / review workspace |
| **Downstream** | Chunk 02: relevance screen & clause routing |
| **Status** | HLD complete · patterns complete · LLD complete · infrastructure complete · test/evaluation complete · CI/CD complete · code not started |
<img width="2752" height="1502" alt="Ingest_and_Integrity_Pipeline_Overview" src="https://github.com/user-attachments/assets/50a856fd-3e89-49bc-be06-702dc4ac66cf" />
[01-Deterministic architecture for precise AI citations](https://notebook.google.com/notebook/dbdfceca-e902-42f0-bb51-04a13b1ff3ba/artifact/246e149b-4746-4f4d-90b2-451b688477a0?utm_source=nlm_web_share&utm_medium=google_oo&utm_campaign=art_share_1&utm_content=&utm_smc=nlm_web_share_google_oo_art_share_1_)

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
