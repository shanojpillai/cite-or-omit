# Chunk 02 · Relevance screen & clause routing

> Keep every potentially relevant clause, route it to the right entitlement family, and make the decision explainable before extraction begins.

| | |
|---|---|
| **Chunk** | 02 of 07 |
| **Upstream** | Chunk 01: ingest, normalise & segment |
| **Downstream** | Chunk 03: evidence-grounded extraction & gap detection |
| **Status** | HLD complete · patterns complete · LLD planned · infrastructure planned · test/evaluation planned · CI/CD planned |

## Reading order

1. [High-level design](hld.md): scope, flow, routing contract and trade-offs.
2. [Patterns](patterns.md): architecture and design patterns behind screening and routing.
3. `lld.md`: planned build-level specification.
4. `infrastructure.md`: planned deployment design.
5. `test-and-evaluation.md`: planned quality and release gates.
6. `ci-cd.md`: planned delivery and operational controls.

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
| Build specification | Planned |
| Infrastructure | Planned |
| Test and evaluation | Planned |
| CI/CD | Planned |
