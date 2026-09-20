# cite-or-omit

> **The model's job is to find and cite, never to fill in.**
> A value with no source span does not exist. A required rule with no source span becomes a question for the customer.

An AI system design case study, worked through as documentation: from an open-ended interview question, through a PRD, a high-level design and a low-level design for each part of the system.

The goal is depth, not coverage. One real-world problem, taken all the way down.

## Quick navigation

- [Repository home](#)
- [PRD](docs/prd/PRD.md)
- [Chunk 01 documentation](docs/chunks/01-ingest/README.md)
- [Chunk 01 HLD](docs/chunks/01-ingest/hld.md)
- [Chunk 01 LLD](docs/chunks/01-ingest/lld.md)
- [Chunk 02 documentation](docs/chunks/02-relevance-screen/README.md)

---

## The question

**Domain:** Workforce management, specifically **time-off entitlements**. These are balances such as PTO, vacation and sick leave. Employees **accrue** hours into a balance and **draw them down** when they take time off.

**User:** An **implementation consultant** at an HCM (Human Capital Management) software vendor. This is not an HR employee. Their job is to read a large pile of a new customer's policy documents and convert them into **business requirements**: what the company intends to give its employees in time off.

**Input:** Customer policy documents such as employee handbooks, HR policies, PTO policies and collective agreements. They arrive as PDF, Word or scanned files, typically **500–600 pages per implementation** for this domain alone.

Most of that content is noise: the CEO's welcome letter, company values, meeting etiquette. The relevant clauses are buried deep in the documents, and every company words them differently. *"PTO"*, *"vacation"*, *"annual leave"* and *"time away from work"* can all mean the same thing, and phrasing varies by region.

**Task:** Design a system that **extracts the entitlement requirements from the documents into a structured format**. The results then go into a **human review and approval workspace**.

**Minimum output per entitlement:**

1. Entitlement type
2. Eligibility rule (e.g., full-time employees, after 90 days)
3. Accrual rule (e.g., 1.54 hrs per bi-weekly pay period, capped at 120 hrs)
4. Policy grouping / bundle it belongs to

**Hard constraints:**

- **Never invent a value.** Every value must come from the source document. If the waiting period isn't stated, the system must not guess "90 days".
- **Detect and flag gaps.** The canonical example is the termination payout clause. Policies often omit it, but configuration requires it, so the system must report it as missing.
- **Every value must be traceable to its source.** This matters for audit, lineage and dispute resolution.

**Simplifying assumptions:**

- All needed information is in the documents. There is no external employee database.
- Messy document parsing (blank pages, complex tables, images) can be named and deferred.
- Storage cost is negligible. Originals are retained permanently.

---

## The seven chunks

The system is decomposed into seven parts. Each chunk gets its own **high-level design**, **patterns**, **low-level design** and **infrastructure design**. Implementation is a future phase.

| # | Chunk | What it does | The hard problem in it |
|---|---|---|---|
| **01** | [**Ingest, normalise & segment**](docs/chunks/01-ingest/README.md) | Accept PDF, Word and scans. Run OCR, render pages, and split documents into sections and clauses, keeping page and coordinates for each. | Pages in must equal pages out. The failures that hurt are the ones no API reports. |
| **02** | **Relevance screen & clause routing** | A cheap, recall-first screen discards the noise. Surviving clauses are routed to entitlement families (vacation, sick, bereavement…) with synonyms and regional terms resolved. | Dropping the one clause that matters is the costliest failure in the system. |
| **03** | **Evidence-grounded extraction & gap detection** | Structured output against the entitlement schema. Every value carries a source span. No span means `not_stated`. Missing required fields become gaps, and disagreements become conflicts. | Absence as a first-class output. Citation verified by deterministic code, not by trusting the model. |
| **04** | **Evaluation harness** | Golden sets built from past implementations. Measures field accuracy, **unsupported-value rate**, **gap recall** and calibration. Gates every change. | What was configured isn't always what the document said. |
| **05** | **Consultant review & approval workspace** | Source page with the span highlighted, beside editable fields. Confidence bands decide review effort. Gaps become customer questions, and every edit is logged. | Fast to verify, hard to rubber-stamp. |
| **06** | **Schema, lexicon & rule packs** | A versioned entitlement schema, a synonym lexicon per region, required-field rules and prompts, all as configuration gated by the harness in CI. | A new region or leave type should be a config change, not a release. |
| **07** | **Infra, scale & failure** | ~600 pages per domain, ~7K pages per customer, processed as batch. Stateless workers sit behind a model gateway. Runs are idempotent and resumable, with full lineage audit. | The bottleneck is model throughput, not pods. |

<img width="3654" height="1182" alt="readme01" src="https://github.com/user-attachments/assets/f0e45e2e-5bc4-463b-bca0-aa3ba8e9e571" />

---

## The journey

```
Question ──► 7 chunks ──► PRD ──► HLD ──► Patterns ──► LLD ──► Infrastructure ──► Test & eval ──► CI/CD
```

| Stage | Output | Status |
|---|---|---|
| 0 · Question & decomposition | This README, the seven chunks | ✅ Done |
| 1 · Product requirements | [`docs/prd/`](docs/prd/): PRD draft and chunk requirements | ✅ In progress |
| 2 · High-level design | [`docs/chunks/`](docs/chunks/): one package per chunk | ✅ Chunks 01–02 |
| 3 · Patterns and rationale | [`docs/chunks/`](docs/chunks/): one package per chunk | ✅ Chunks 01–02 |
| 4 · Low-level design | [`docs/chunks/`](docs/chunks/): one package per chunk | ✅ Chunks 01–02 |
| 5 · Infrastructure | [`docs/chunks/`](docs/chunks/): one package per chunk | ✅ Chunks 01–02 |
| 6 · Test and evaluation | [`docs/chunks/`](docs/chunks/): one package per chunk | ✅ Chunks 01–02 |
| 7 · CI/CD | [`docs/chunks/`](docs/chunks/): one package per chunk | ✅ Chunks 01–02 |
| 8 · Implementation | Future phase outside this documentation workspace | ⬜ Not started |

### Chunk progress

| Chunk | HLD | LLD | Infra | Test / Eval | CI/CD | Code |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| [01 Ingest, normalise & segment](docs/chunks/01-ingest/README.md) | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| [02 Relevance screen & routing](docs/chunks/02-relevance-screen/README.md) | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| 03 Cited extraction & gaps | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 04 Evaluation harness | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 05 Review & approval workspace | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 06 Schema, lexicon & rule packs | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 07 Infra, scale & failure | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |

---

## Repository layout

```
cite-or-omit/
├── README.md                               ← you are here
├── LICENSE                                 ← MIT open-source license
├── docs/
│   ├── prd/
│   │   ├── PRD.md                          ← product requirements draft
│   ├── chunks/                             ← one complete documentation package per chunk
│   │   └── 01-ingest/
│   │       ├── README.md                   ← scope, status and document map
│   │       ├── hld.md                      ← high-level design
│   │       ├── patterns.md                 ← patterns and rationale
│   │       ├── lld.md                      ← low-level design
│   │       ├── infrastructure.md          ← deployment design
│   │       ├── test-and-evaluation.md     ← testing and quality evaluation
│   │       └── ci-cd.md                   ← release and deployment controls
└── .gitignore
```

---

## Conventions

**Every design doc follows the same shape.**
1. The one-line claim
2. Numbered data flow (diagram + table)
3. How it evolved: what was tried first and why it broke
4. Trade-offs: what this bought, what it cost
5. Failure modes and how they're detected
6. Open questions

**Source material.** Any future fixtures must be synthetic or public policy text. No real customer documents, ever.

---

## Design principles

In priority order. When a design choice is ambiguous, these decide it.

1. **Cite or omit.** Every extracted value carries a pointer to its source span. No span, no value.
2. **Absence is an output.** `not_stated` is a first-class state. For a required rule, it becomes a gap.
3. **The consultant approves; the system proposes.** Nothing leaves without a named human approving it.
4. **Recall when finding, precision when stating.** Keep too much at the screen, say too little at extraction.
5. **Measured before trusted.** No model, prompt or rule change ships without passing the evaluation gate.
6. **Domain rules live in configuration, not code.** Experts own the schema, lexicon and gap rules, versioned.

---

## A worked example

**Source text:**
> *"Regular full-time employees become eligible for Paid Time Off after completing ninety (90) days of continuous service. PTO accrues at 1.54 hours per bi-weekly pay period. Employees may not accrue more than 120 hours."*

**Extracted:**

| Field | Status | Value | Cited text |
|---|---|---|---|
| Entitlement type | extracted | Vacation / PTO | "Paid Time Off" |
| Policy bundle | extracted | Regular full-time | "Regular full-time employees" |
| Waiting period | extracted | 90 calendar days | "ninety (90) days of continuous service" |
| Accrual rate | extracted | 1.54 hrs / bi-weekly period | "1.54 hours per bi-weekly pay period" |
| Balance cap | extracted | 120 hrs | "may not accrue more than 120 hours" |
| Carryover | **not stated** | — | — |
| Termination payout | **not stated → GAP** | — | — |

**Generated customer question:**
> *"Your handbook (p. 31) doesn't say what happens to unused PTO when an employee leaves. Is it paid out, forfeited, or handled per state law?"*

---

## About

A self-directed documentation deep dive into designing trustworthy LLM systems for regulated, high-consequence domains, where a wrong value lands on someone's pay stub.

The problem is a generic HCM implementation use case. It does not describe any specific vendor's internal systems.

**Author:** Shanoj Kumar V · [shanoj.com](https://shanoj.com) · [github.com/shanojpillai](https://github.com/shanojpillai)

This repository is licensed under the [MIT License](LICENSE).
