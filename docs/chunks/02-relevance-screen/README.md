# Chunk 02 · Relevance screen & clause routing

> Keep every potentially relevant clause, route it to the right entitlement family, and make the decision explainable before extraction begins.

| | |
|---|---|
| **Chunk** | 02 of 07 |
| **Upstream** | Chunk 01: ingest, normalise & segment |
| **Downstream** | Chunk 03: evidence-grounded extraction & gap detection |
| **Status** | HLD complete · patterns complete · LLD complete · infrastructure complete · test/evaluation complete · CI/CD complete |

## Reading order

1. [High-level design](hld.md): scope, flow, routing contract and trade-offs.
2. [Patterns](patterns.md): architecture and design patterns behind screening and routing.
3. [Low-level design](lld.md): build-level specification.
4. [Infrastructure](infrastructure.md): AWS deployment design.
5. [Test and evaluation](test-and-evaluation.md): quality and release gates.
6. [CI/CD](ci-cd.md): delivery and operational controls.

## Boundary

This chunk receives canonical segments from Chunk 01, screens them with recall-first logic, routes relevant clauses to entitlement families, resolves cross-references, and groups related evidence. It does not extract entitlement values, detect required-field gaps, approve records, or own the source document model.

## Contracts

- **Input:** Chunk 01 `document.ready/v1` and canonical `Document -> Page -> Segment -> Anchor` records.
- **Output:** versioned relevance and routing results for Chunk 03, with the original segment ids and citations preserved.
- **Quality priority:** false negatives are more costly than false positives; discarded content remains auditable.

## Document map

| Concern | Canonical location |
|---|---|
| Architecture | [HLD](hld.md) |
| Patterns and rationale | [Patterns](patterns.md) |
| Build specification | [LLD](lld.md) |
| Infrastructure | [Infrastructure](infrastructure.md) |
| Test and evaluation | [Test and evaluation](test-and-evaluation.md) |
| CI/CD | [CI/CD](ci-cd.md) |
