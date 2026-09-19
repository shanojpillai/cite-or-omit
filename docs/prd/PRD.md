*Product Requirements Document · Services Transformation · Implementation AI*

# Entitlement Extraction Assistant

Turning customer time-off policy documents into cited, reviewable entitlement requirements so implementation consultants validate instead of transcribing.

| | |
|---|---|
| **Status** | `Draft v0.4 · in review` |
| **Author** | Shanoj Kumar V (practice document) |
| **Last updated** | 19 Sep 2026 |
| **Target release** | Alpha Q4 2026 · GA Q2 2027 |
| **Reviewers** | Eng Lead · AI/ML Lead · Design · Services Ops · Privacy · Legal |
| **Approvers** | Director, Services Transformation · VP Services |

> **Context.** This is a practice PRD written for system-design preparation. It describes a generic HCM vendor's Services organisation; it is not an official document of any company. Figures tagged `ASSUMPTION` are working hypotheses to validate in discovery before this PRD is approved.

## Related design documents

This PRD is the parent artifact for the chunk-level implementation design. Use the links below to move from the end-to-end requirements into the detailed design for the first operational slice:

- [Chunk 01 · Ingest, normalise & segment](Chunk%2001%20%C2%B7%20Ingest,%20normalise%20&%20segment.md)
- [Chunk 01 · Ingest, normalise & segment — Low-Level Design](Chunk%2001%20%C2%B7%20Ingest,%20normalise%20&%20segment%20%E2%80%94%20Low-Level%20Design.md)

## 01. TL;DR

Implementation consultants spend a large share of every implementation reading customer handbooks and policy documents to work out what time off the customer gives its employees. They then write those rules up as business requirements that drive entitlement configuration. The work is slow and repetitive, its quality depends on who does it, and nothing traces a requirement back to the document it came from. When a clause is misread, the error surfaces weeks later as a wrong balance on an employee's pay stub.

The **Entitlement Extraction Assistant** ingests the customer's documents, finds the clauses that govern time off, and extracts each entitlement into a structured record. **Every field is cited to the exact span of source text it came from.** Required rules the documents do not state are flagged as gaps instead of being guessed. A consultant reviews the source and the extraction side by side, corrects or confirms each field, turns gaps into questions for the customer, and approves a requirements package that feeds configuration.

> **The model's job is to find and cite, never to fill in. A value with no source span does not exist, and a required rule with no source span becomes a question for the customer.**<br>*This is the single rule every requirement in this document is checked against.*

**What we ship first (MVP)** Vacation/PTO and sick leave · US and Canada · English · PDF, DOCX and scanned PDF · consultant review workspace · requirements export.

**How we'll know it worked** Consultant hours for the entitlement discovery phase down ≥40% · zero unsupported values in approved requirements · ≥95% of missing required rules flagged before customer sign-off.

## 02. Problem & evidence

### 2.1 The problem

Every new customer arrives with a stack of policy material: an employee handbook, standalone PTO and sick policies, collective agreements for unionised groups, regional addenda and offer-letter templates. For the entitlement domain alone, a consultant typically works through **500–600 pages per implementation**. Most of it is irrelevant: welcome letters, company values, codes of conduct, meeting etiquette. The rules that matter sit in a few scattered clauses, written in each company's own language. "PTO", "vacation", "annual leave", "time away from work" and "congé annuel" can all mean the same thing.

Consultants have to find those clauses, interpret them, write them down as structured requirements, and also notice what the documents *don't* say. The rule most often missing is what happens to an unused balance at termination. Policies rarely state it, yet configuration needs it.

### 2.2 Why it matters

| Dimension | Impact today | Evidence |
|---|---|---|
| Cost to serve | Discovery and requirements for entitlements take roughly 25–40 consultant hours per implementation, repeated across 10–12 policy domains. | `ASSUMPTION` validate via time-tracking codes |
| Time to go-live | Requirements sign-off is on the critical path. Missed gaps found late trigger rework cycles with the customer. | `ASSUMPTION` sample 20 recent projects |
| Quality & risk | Configuration errors show up as wrong accruals or payouts on employee pay stubs, which leads to escalations, remediation and possible statutory non-compliance. | Post-go-live defect tickets tagged "entitlement" |
| Consistency | Two consultants read the same clause differently. Interpretation lives in personal notes, not in a reviewable artifact. | Consultant interviews (discovery) |
| Auditability | No link from a configured rule back to the policy text that justified it. Disputes get settled from memory. | Services Ops feedback |

### 2.3 Why now

- Current LLMs reliably produce schema-conformant structured output and can quote source spans, which makes citation-grounded extraction practical at an acceptable cost.
- Services Transformation has a mandate to cut implementation hours. Entitlements is a narrow, high-volume, well-bounded domain, which makes it a good first proof point for the pattern the other domains will reuse.
- Past implementations give us an answer key: historical policy documents paired with the entitlements that were actually configured. We can measure quality *before* any consultant relies on the output.

## 03. Users & personas

| Persona | Role in the workflow | What they need | What they fear |
|---|---|---|---|
| **Implementation Consultant**<br>`primary` | Reads customer documents, writes entitlement requirements, runs the customer sign-off. | Relevant clauses found fast. A draft they can trust, then verify in seconds. A list of what to ask the customer. | A tool that invents rules they then have to defend to the customer, and more review work than it saves. |
| **Solution Lead / Senior Consultant** | Reviews requirements before customer sign-off. Owns quality across projects. | Consistency across consultants. Visibility into low-confidence fields and unresolved gaps. | Junior consultants rubber-stamping AI output. |
| **Customer HR / Payroll Admin**<br>`indirect` | Answers gap questions and signs off on requirements. | Clear, specific questions that quote their own policy back to them. | Being asked the same thing twice, or signing something they don't recognise. |
| **Domain SME / Prompt Owner** | Maintains the entitlement schema, the synonym lexicon, gap rules and extraction instructions. | Change rules safely, with a regression check before anything reaches production. | One edit silently breaking another entitlement type. |
| **Services Ops / Compliance** | Audit, dispute resolution, process metrics. | For any configured rule: which document, which clause, who approved it, when. | Unexplainable configuration in a regulated pay context. |

## 04. Current-state journey (entitlements only)

| # | Step today | Est. hours | Pain | Future state |
|---|---|---|---|---|
| 1 | Collect documents from customer by email or shared drive | 1–2 | Versions unclear, duplicates | Upload to a project workspace with de-duplication by content hash |
| 2 | Skim everything to find time-off content | 6–10 | ~85% of pages irrelevant | Relevance screen hides the noise and shows only candidate clauses |
| 3 | Interpret each clause into rules | 8–12 | Synonyms, regional wording, tables, nested exceptions | Pre-filled entitlement records, each field cited |
| 4 | Write requirements in the workbook | 4–6 | Re-typing, no link to source | Approved records export directly |
| 5 | Spot what's missing, email the customer | 2–4 | Depends on experience, often late | Gap list generated against required rules, with draft questions |
| 6 | Lead review, customer sign-off | 3–5 | Reviewer re-reads source to check | Reviewer sees citations and low-confidence flags |
|  | **Total** | **24–39** | `ASSUMPTION` baseline to be measured in a 2-week time-and-motion study with 6 consultants |  |

## 05. Goals & non-goals

### 5.1 Goals

| # | Goal | Type | Measure |
|---|---|---|---|
| G1 | Cut consultant effort for entitlement discovery and requirements | Business | Hours per implementation, steps 2–5: −40% at GA, −60% stretch |
| G2 | Never introduce a value that isn't in the customer's documents | Trust | Unsupported-value rate in approved packages = 0; in raw model output ≤0.5% of fields |
| G3 | Surface missing required rules before customer sign-off | Quality | Gap recall ≥95% on the golden set |
| G4 | Make every configured entitlement traceable to its source | Compliance | 100% of approved fields carry a document, page and span reference, plus approver and timestamp |
| G5 | Earn consultant adoption, not mandate it | Adoption | ≥70% of eligible new implementations use it by GA+1 quarter; consultant CSAT ≥4.0/5 |

### 5.2 Non-goals (v1)

| Non-goal | Why not now |
|---|---|
| Writing configuration directly into the product (auto-configure) | The human-approved requirement is the trust boundary. Mapping requirements to configuration is a separate initiative that consumes this output. |
| Deciding statutory minimums or legal compliance | We extract what the *customer's policy says*. Checking it against employment standards is a legal product with different liability. P2: optional advisory flag. |
| Other policy domains (overtime, pay rules, scheduling) | Prove the pattern on entitlements first. The architecture must not preclude them (see P2). |
| Customer self-service upload and review | v1 users are internal consultants. Customer-facing surfaces change security, UX and support scope. |
| Languages other than English | French-Canadian is P1, gated on golden-set coverage. Others later. |
| Chat assistant over the documents | Useful, but it is a different job. The extraction contract has to be solid first. |

## 06. Product principles

When a requirement is ambiguous, these decide it. In priority order:

1. **Cite or omit.** Every extracted value carries a pointer to its source span. No span means no value. The system never fills a default, infers from industry norms, or carries a rule over from another customer.
2. **Absence is an output.** "Not stated in the documents" is a first-class state, as visible as an extracted value. For a required rule it becomes a gap.
3. **The consultant approves; the system proposes.** Nothing leaves the tool without a named human approving it. The UI is designed so checking takes seconds, and so it is hard to approve without looking.
4. **Recall before precision in finding, precision before recall in stating.** Dropping a relevant clause is the costliest failure, so the screen errs toward keeping. Stating a wrong value is the most dangerous failure, so extraction errs toward "not stated".
5. **Measured before trusted.** No model, prompt or rule change reaches consultants without passing the evaluation gate against the golden set.
6. **Domain rules live in configuration, not code.** SMEs own the schema, lexicon and gap rules as versioned configuration. Engineering owns the pipeline.

## 07. Scope & phasing

| Phase | Scope | Users | Exit criteria |
|---|---|---|---|
| **Phase 0**<br>Foundations<br>6 wks | Golden set (≥40 historical implementations), entitlement schema v1, eval harness, baseline time study | AI team + 2 SMEs | Harness produces all metrics in §13. Baseline hours measured. |
| **Phase 1**<br>Alpha — shadow<br>8 wks | MVP pipeline + review workspace. Consultants still do the work the usual way, and compare. | 6–8 consultants, live projects | Field accuracy ≥90%, unsupported ≤0.5%, gap recall ≥90%. No P0 defects. |
| **Phase 2**<br>Beta — assisted<br>10 wks | Consultants use the output as their draft. Export to requirements workbook. Customer question drafts. | 2 regional teams (~30 consultants) | Hours −30% vs baseline, CSAT ≥3.8, zero unsupported values in approved packages |
| **GA** | All new US/CA implementations. French-Canadian if gated metrics pass. | All consultants | G1–G4 met for 4 consecutive weeks |
| **Next** | Additional leave types, further domains (overtime, holidays, pay rules) reusing the pipeline with new rule packs | — | Separate PRD per domain |

#### MVP entitlement types

`P0` Vacation / PTO / annual leave `P0` Sick leave (incl. statutory references as stated) `P1` Personal / floating days `P1` Bereavement `P2` Parental, jury duty, volunteer, and others

## 08. Entitlement domain model

This is the contract between extraction, review and export. One document set produces **many entitlement records**: one per entitlement type × eligible population. Each field value has the same envelope.

### 8.1 Field envelope (applies to every field)

| Attribute | Type | Meaning |
|---|---|---|
| `status` | enum | `extracted` · `not_stated` · `conflicting` · `ambiguous` |
| `value` | typed | Normalised value. Null unless status = extracted. |
| `verbatim` | string | The exact source text the value was taken from |
| `citations[]` | list | `doc_id`, `doc_version`, `page`, `char_start/char_end`, `bbox`. At least one is required when status = extracted. |
| `confidence` | 0–1 | Calibrated against reviewer outcomes. The raw model score is never displayed. |
| `derivation` | string? | Present only when the value was computed from stated values (e.g., 10 days × 8 hrs). Shows the formula and its inputs. |
| `review` | object | `state` (pending / confirmed / edited / rejected), reviewer, timestamp, original value if edited, reason code |

### 8.2 Entitlement record fields

| Field | Example | Required? | If missing |
|---|---|---|---|
| `entitlement_type` | Vacation (source term: "Paid Time Off") | `yes` | — |
| `policy_bundle` | "Full-time salaried – US" | `yes` | Gap |
| `eligibility.population` | Full-time, ≥30 hrs/week | `yes` | Gap |
| `eligibility.waiting_period` | 90 calendar days | `yes` | Gap. May legitimately be "none stated", so ask to confirm. |
| `accrual.method` | Per pay period · front-loaded · per hour worked | `yes` | Gap |
| `accrual.rate` + `unit` | 1.54 hours / biweekly period | `yes` | Gap |
| `accrual.tiers[]` | 0–4 yrs: 10 d; 5+ yrs: 15 d | conditional | Gap if tenure-based language is present but incomplete |
| `balance.cap` | Max 120 hours | `yes` | Gap |
| `carryover` | Up to 40 hrs to next year | `yes` | Gap |
| `year_basis` | Calendar · anniversary · fiscal | `yes` | Gap |
| `termination_payout` | Paid out · forfeited · per statute | `yes` | Gap. Most frequently missing field. |
| `negative_balance` | Up to −16 hrs | `recommended` | Warning |
| `usage_rules` | Min increment 1 hr; 2 weeks' notice | `optional` | — |
| `jurisdiction` | US-CA; CA-ON | `yes` | Gap |

Required-field rules are **configuration**: a rule pack per entitlement type and jurisdiction that SMEs own (FR-CFG-02). Some jurisdictions add required fields; sick leave in some regions, for example, requires a statutory accrual reference.

## 09. User stories

#### Implementation Consultant

1. As a consultant, I want to upload a customer's whole document set at once so I don't have to sort it first.
2. As a consultant, I want the system to set aside pages that don't govern time off so I only read what matters, and I want to be able to see what it set aside.
3. As a consultant, I want a draft entitlement record per leave type and employee group, with every value linked to the sentence it came from, so I can verify each one in seconds.
4. As a consultant, I want fields the documents don't cover to be clearly marked "not stated" rather than filled in, so I never defend a rule the customer didn't write.
5. As a consultant, I want to see where two documents contradict each other (handbook says 10 days, PTO policy says 12) so I can resolve it with the customer.
6. As a consultant, I want draft customer questions for every gap and conflict, each quoting their own policy, so I can send one clean email.
7. As a consultant, I want to edit a value and have my edit recorded with a reason, so reviewers see what changed and the system learns where it's weak.
8. As a consultant, I want to export approved requirements in the format our requirements workbook already uses, so nothing downstream changes.
9. As a consultant, when the customer sends a revised policy, I want to see only what changed since my last approval so I don't re-review everything.

#### Solution Lead

10. As a solution lead, I want a project-level view of unresolved gaps, low-confidence fields and edits, so I can focus my review where risk is.
11. As a solution lead, I want to see who approved each field and when, so accountability is clear before customer sign-off.

#### Domain SME / Prompt Owner

12. As an SME, I want to add a synonym ("congé annuel" = vacation) or a required-field rule without an engineering release, with the change evaluated automatically before it goes live.
13. As an SME, I want per-field accuracy broken down by entitlement type and jurisdiction, so I know which rule to fix next.

#### Services Ops / Compliance

14. As compliance, given any configured entitlement, I want to retrieve the source clause, document version, model and prompt versions, and approver, so disputes can be settled from the record.

#### Edge cases the stories must cover

- A document with no time-off content at all: say so explicitly, don't return an empty record.
- Rules in tables (tenure → days) rather than prose.
- Different rules for different populations in one clause (full-time vs part-time, union vs non-union).
- Rules expressed in days when the target unit is hours, with no hours-per-day stated. That is a gap, not a conversion.
- A scanned document with poor OCR quality: flag the page, don't extract low-quality text silently.
- A policy that defers to law ("as required by applicable law"). Extract it as a stated reference, not as a value.
- Superseded documents in the set (a 2022 handbook alongside a 2025 update).

## 10. Functional requirements

These are organised by pipeline stage. Priority: `P0` required for alpha, `P1` required for GA, `P2` future, design for it.

`Ingest & segment → Relevance screen & routing → Cited extraction & gaps → Consultant review → Export · Config & rule packs Eval harness Audit & ops`

### 10.1 Ingest, normalisation & segmentation

| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-ING-01 | Accept PDF (text and scanned), DOCX, DOC and TIFF/PNG uploads, up to 200 files and 2,000 pages per project batch. | `P0` | • Unsupported types are rejected at upload with the file name and the reason<br>• Password-protected files are flagged and never silently skipped |
| FR-ING-02 | De-duplicate by content hash. Detect likely versions of the same document (title and similarity) and ask the consultant which one is current. | `P1` | • An identical re-upload creates no new processing<br>• Superseded documents are excluded from extraction but kept for audit |
| FR-ING-03 | Render every page to a stored image and extract a text layer. Run OCR where no text layer exists or its quality is below threshold. | `P0` | • Pages in = pages out, asserted per document; a mismatch fails the document loudly<br>• Pages with OCR confidence below threshold are flagged in the review UI |
| FR-ING-04 | Segment into a structure of sections, clauses, lists and tables. Each segment keeps its page, character offsets and bounding box. | `P0` | • Tables keep row/column structure<br>• Each segment's text maps back to page coordinates for highlighting |
| FR-ING-05 | Keep originals immutable for the retention period. Any derived artefact references the original by hash and version. | `P0` | An original can be re-fetched byte-identical for any approved field |

### 10.2 Relevance screen & routing

| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-SCR-01 | Classify each segment as *relevant to entitlements* or not, tuned for recall. Use a low-cost model plus lexicon hints. | `P0` | • Segment-level relevance recall ≥98% on the golden set<br>• Discarded segments stay browsable ("Show hidden pages") |
| FR-SCR-02 | Route relevant segments to one or more entitlement families (vacation, sick, personal…). Resolve synonyms and regional terms through the SME lexicon. | `P0` | "Time away from work", "annual leave" and "PTO" all route to Vacation/PTO in the test suite |
| FR-SCR-03 | Group segments that describe the same entitlement across documents, and detect cross-references ("see Appendix B"). | `P1` | Referenced appendix tables are pulled into the same extraction context |
| FR-SCR-04 | Report a document with no relevant content explicitly: "No time-off rules found in *Code of Conduct.pdf*". | `P0` | Shown in the project summary, never as an empty record |

### 10.3 Cited extraction & gap detection

| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-EXT-01 | Extract entitlement records against the schema (§8) using constrained structured output. One record per entitlement type × population. | `P0` | 100% of outputs validate against the schema. Invalid output is retried once, then marked failed. It is never "repaired" by inventing values. |
| FR-EXT-02 | **Citation required.** Every `extracted` field includes the verbatim span and its location. | `P0` | Any field without a citation is downgraded to `not_stated` before it is shown |
| FR-EXT-03 | **Citation verification.** A deterministic check confirms the verbatim text exists at the cited location (normalised match), and that the value is consistent with the verbatim text (numbers and units appear in the span). | `P0` | • A failed check downgrades the field to `ambiguous` with reason "citation not verified"<br>• Verification failure rate is reported per run |
| FR-EXT-04 | Gap detection: evaluate each record against the rule pack's required fields. Any required field in `not_stated` becomes a gap with a reason. | `P0` | Gap recall ≥95% on the golden set; every gap names the missing field and the entitlement it belongs to |
| FR-EXT-05 | Conflict detection: when two sources state different values for the same field and population, mark it `conflicting` with both citations. | `P0` | No conflict is silently resolved by picking one value |
| FR-EXT-06 | Derived values (e.g., days → hours) are allowed only when every input is itself cited. The derivation is shown. | `P1` | A derivation with an uncited input is rejected and the field becomes a gap |
| FR-EXT-07 | A calibrated confidence per field, mapped to review bands: *High* (sample review), *Medium* (review), *Low* (review required, highlighted). | `P1` | In each band, observed accuracy is within ±5 pts of the band's stated accuracy |
| FR-EXT-08 | Draft a customer question for each gap and conflict, quoting the relevant policy text. | `P1` | Consultants rate ≥80% of questions "send as-is or minor edit" in beta |

### 10.4 Review & approval

| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-REV-01 | Side-by-side view: source page with the cited span highlighted on the left, editable field on the right. | `P0` | Selecting a field scrolls to and highlights its span in under 300 ms |
| FR-REV-02 | Per-field actions: Confirm, Edit (reason required), Reject, Mark as gap. For Edit, the reviewer can select a different source span as the new citation. | `P0` | An edited value with no citation is allowed only with reason "customer-confirmed", which requires attaching the customer response |
| FR-REV-03 | A record can be approved only when every required field is confirmed, edited, or has a recorded customer answer. | `P0` | The Approve button is disabled with a count of what's outstanding |
| FR-REV-04 | Gaps & questions panel: a consolidated list, exportable as a customer email or document, with answers captured back into the record. | `P1` | A captured answer fills the field with citation type `customer_response` |
| FR-REV-05 | Change review: on re-upload of a revised document, show only changed fields and newly detected clauses since the last approval. | `P1` | Unchanged approved fields keep their approval state |
| FR-REV-06 | Lead view: project summary of records, gaps, conflicts, low-confidence fields and edit rate, with approver per record. | `P1` | Filterable by consultant and state |

### 10.5 Export

| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-EXP-01 | Export approved records to the existing requirements workbook template (XLSX), with a citation column per field. | `P0` | Solution leads confirm the output needs no reformatting |
| FR-EXP-02 | Versioned JSON export via API for the downstream configuration-mapping initiative. | `P1` | Schema published and versioned. Breaking changes need a major version. |
| FR-EXP-03 | Only approved records export. Draft exports carry a watermark. | `P0` | — |

### 10.6 Configuration & rule packs

| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-CFG-01 | Entitlement schema, synonym lexicon, required-field rules and extraction instructions live as versioned configuration in source control, with an owner per pack. | `P0` | Every run records the exact versions it used |
| FR-CFG-02 | Rule packs per entitlement type × jurisdiction, with inheritance (base → country → province/state). | `P1` | Adding a province is a configuration change with no code change |
| FR-CFG-03 | Any configuration or model change triggers a scoped evaluation run. Merge is blocked on regression beyond tolerance. | `P0` | Pass/fail check on the change request within 20 minutes for single-pack changes |

### 10.7 Audit & operations

| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-AUD-01 | Immutable audit log of every extraction, edit, approval and export: actor, time, before/after, model, prompt and config versions. | `P0` | Lineage for any approved field is retrievable in one query |
| FR-AUD-02 | Runs are idempotent and resumable. A failed page or document does not fail the batch. | `P0` | Re-running a completed batch with unchanged inputs and versions costs no model calls |
| FR-AUD-03 | Per-run cost and token accounting, by project and stage. | `P1` | Cost per implementation visible on the ops dashboard |

## 11. Review workspace UX

Review is where the product earns trust or loses it. The design goal is **fast to verify, hard to rubber-stamp**.

**Layout**

- **Left:** the source page image with the cited span boxed and the clause context shaded
- **Right:** entitlement records as cards; fields as rows with status chip, value, confidence band
- **Top bar:** project progress (records approved / total, open gaps, conflicts)

**Field states (chip + colour + icon, never colour alone)**

- `Extracted · High` sample review
- `Extracted · Low` review required
- `Gap` required, not stated
- `Conflict` sources disagree
- `Not stated` optional field

#### Anti-rubber-stamp design

- No "approve all" at project level. Approval is per record, and low-band fields need an explicit per-field confirm.
- Low-band fields open with the source highlighted and the value hidden until the reviewer has had the clause in view. `ASSUMPTION` test in alpha; it reduces anchoring on the model's answer.
- A 5% random sample of High-band fields is routed for mandatory review. The measured accuracy on that sample feeds calibration (§13).
- Edit reason codes: *wrong value · wrong span · wrong population · misread table · other*. These drive the SME error backlog.

#### Empty & error states

- No relevant content found: "We didn't find time-off rules in these 14 documents. Check that the handbook or PTO policy is included." Include a link to the hidden pages.
- Processing a page failed: the page is listed with its reason (unreadable scan, encrypted) and a *Retry* or *Mark as reviewed manually* action.
- Accessibility: WCAG 2.1 AA, full keyboard review flow (J/K next field, C confirm, E edit).

## 12. Non-functional requirements

| Area | Requirement | Target |
|---|---|---|
| Throughput | End-to-end batch for a typical project (600 pages) | ≤30 min p90, ≤60 min p99 |
| Interactive latency | Field select → span highlight; save edit | ≤300 ms / ≤500 ms p95 |
| Cost | Model spend per implementation (entitlements) | ≤$15 at GA `ASSUMPTION`, alert at 2× the rolling median |
| Scale | Concurrent projects at peak season | 150 active projects, ~90K pages/day `ASSUMPTION` |
| Availability | Review workspace | 99.5% business hours. The batch pipeline may queue during a provider outage and must not lose work. |
| Resilience | Model provider failure | Retries with backoff, circuit breaker, fallback to a secondary provider only with an equivalently evaluated configuration |
| Security | Customer documents may contain employee PII (names in examples, signatures) | Encryption in transit and at rest; tenant-isolated storage; access limited to the project team; SSO and RBAC |
| Privacy | Model provider data handling | Zero-retention / no-training terms; region-pinned inference for Canadian customers where contractually required |
| Retention | Originals, derived artefacts, audit log | Follow customer contract. Audit log ≥7 years `ASSUMPTION` confirm with Legal. |
| Observability | Per-stage traces, token counts, latency, verification failures, gap and edit rates | Dashboards plus alerting on quality drift |
| Accessibility | Review workspace | WCAG 2.1 AA |

## 13. AI quality & evaluation

The evaluation harness is a product requirement, not an engineering nice-to-have. It is built in Phase 0, before the pipeline is tuned, and it gates every release.

### 13.1 Golden set

- **Source:** ≥40 historical implementations with the original policy documents and the requirements actually configured and signed off. Target 100 by GA.
- **Coverage:** stratified by entitlement type, jurisdiction (≥8 US states, ≥4 provinces), document quality (native vs scanned), unionised vs not, table-heavy vs prose.
- **Labelling:** SMEs re-annotate each field with its source span and mark gaps. Historical configurations sometimes contain values the customer supplied verbally, so "configured" is not the same as "in the documents". Double annotation on 20%, with inter-annotator agreement reported.
- **Adversarial slice:** documents with deliberate omissions and contradictions, to measure gap and conflict detection.
- **Hygiene:** customer data is anonymised under Privacy approval. The golden set is versioned; results cite the version.

### 13.2 Metrics & release gates

| Metric | Definition | Alpha gate | GA gate |
|---|---|---|---|
| Relevance recall | Relevant segments kept ÷ all relevant segments | ≥97% | ≥98% |
| Field accuracy | Fields whose value and status both match gold ÷ gold fields | ≥88% | ≥93% |
| **Unsupported-value rate** | Extracted values with no support in the source ÷ extracted values | ≤0.5% | ≤0.2% |
| Citation validity | Citations whose span exists and supports the value | ≥97% | ≥99% |
| **Gap recall** | Correctly flagged gaps ÷ gold gaps | ≥90% | ≥95% |
| Gap precision | True gaps ÷ flagged gaps (false gaps waste customer goodwill) | ≥75% | ≥85% |
| Conflict recall | On the adversarial slice | ≥80% | ≥90% |
| Calibration | Expected calibration error across bands | ≤0.08 | ≤0.05 |
| Cost / latency | Per 600-page project | tracked | NFR targets |

Scoring is **per field**, broken down by entitlement type, jurisdiction and document quality. An aggregate score hides the slice that hurts customers. Matching uses exact comparison for enums and numbers after unit normalisation, and a reviewed LLM judge only for free-text fields, with the judge itself spot-checked by SMEs.

### 13.3 Online quality loop

- Every consultant edit is a labelled error with a reason code. The weekly error review turns the top clusters into rule-pack or prompt changes, which pass the gate before release.
- Drift alerts fire if the weekly edit rate or verification-failure rate moves more than 25% from its trailing 4-week mean.
- Model upgrades run as a side-by-side evaluation on the full golden set. The swap is a configuration change, approved by the AI Lead against these gates.

## 14. Success metrics

**North-star:** consultant hours to produce signed-off entitlement requirements, per implementation.

| Metric | Kind | Baseline | Target (GA+1Q) | Source |
|---|---|---|---|---|
| Hours, entitlement discovery → sign-off | Lagging · north star | 24–39 `ASSUMPTION` | −40% (stretch −60%) | Time-tracking codes |
| Unsupported values in approved packages | Guardrail | n/a | 0 | Audit sampling (5% of approved fields monthly) |
| Post-go-live entitlement defects | Lagging | to measure | −30% | Defect tracker, "entitlement" tag |
| Customer question rounds per implementation | Lagging | to measure | −1 round | Project records |
| Adoption | Leading | 0 | ≥70% of eligible projects | Product analytics |
| Edit rate per field | Leading | alpha measure | ≤10% | Review events |
| Median review time per record | Leading | alpha measure | ≤4 min | Review events |
| Consultant CSAT | Leading | — | ≥4.0 / 5 | In-product survey after sign-off |
| Rubber-stamp signal | Guardrail | — | High-band sample error caught ≥ injected rate | Canary fields (see R3) |

## 15. Risks & mitigations

| # | Risk | Likelihood / Impact | Mitigation |
|---|---|---|---|
| R1 | Model states a plausible value the document doesn't contain | Med / **High** | Citation-required schema, deterministic span verification (FR-EXT-03), unsupported-value gate, human approval |
| R2 | Relevance screen drops the one clause that matters | Low / High | Recall-tuned screen, hidden pages stay browsable, recall gate ≥98%, gap detection as a second net |
| R3 | Consultants over-trust and rubber-stamp | Med / High | Per-record approval, forced review of low band, random High-band sampling, periodic canary fields with known errors (disclosed to consultants as part of QA) |
| R4 | Golden set reflects what was configured, not what the documents say | High / Med | SME re-annotation with spans; separate "customer-supplied" label |
| R5 | Consultants see the tool as a threat or extra work, and adoption stalls | Med / High | Consultants in design from Phase 0, shadow-mode alpha, measure and publish time saved, no mandate before beta results |
| R6 | Customer data handling objections (PII, provider terms) | Med / Med | Privacy review in Phase 0, zero-retention terms, region pinning, contract language update |
| R7 | Too many false gaps annoy customers | Med / Med | Gap precision gate, consultant curates questions before sending |
| R8 | Scanned or low-quality documents degrade accuracy | High / Med | OCR quality flag per page, low-quality pages routed to manual review, quality slice tracked in eval |
| R9 | Cost growth as domains are added | Med / Low | Cheap-model screen before expensive extraction, caching by content hash, per-stage cost accounting |

## 16. Dependencies

| Dependency | Owner | Needed by | Status |
|---|---|---|---|
| Access to historical implementation documents + configurations for golden set | Services Ops + Privacy | Phase 0 wk 1 | `blocking` |
| SME time: 2 senior consultants at 50% for Phase 0–1 | Services leadership | Phase 0 | `requested` |
| Approved LLM providers with enterprise terms | Procurement / Security | Phase 0 | `in place `ASSUMPTION`` |
| Requirements workbook template (current version) | Services Methodology | Phase 1 | `available` |
| SSO / RBAC integration with project staffing system | Platform team | Phase 1 | `to scope` |
| Config-mapping initiative (consumer of JSON export) | Adjacent team | GA | `aligned on schema` |

## 17. Rollout & launch gates

| Gate | Criteria to pass | Decision owner |
|---|---|---|
| Enter alpha | Golden set ≥40; alpha metric gates in §13 met offline; Privacy & Security sign-off; consultant training done | AI Lead + Director |
| Enter beta | 6 alpha projects complete in shadow; no P0 defects open; alpha gates hold on live data; CSAT ≥3.5 | Director, Services Transformation |
| GA | G1–G4 met 4 consecutive weeks in beta; runbook and on-call in place; support trained | VP Services |

#### Kill / pause criteria

- Any unsupported value reaches a customer-signed package: pause new projects, root-cause, re-gate.
- Beta hours reduction <15% after 10 weeks: stop and re-scope. The value hypothesis failed.
- Edit rate >25% sustained: extraction is not ready; return to Phase 1.

#### Enablement

A 45-minute consultant training covers what the tool does and doesn't do, reading citations, and when to reject. Office hours run weekly through beta. There is a feedback channel with a named product owner.

## 18. Open questions

| # | Question | Owner | Blocking? |
|---|---|---|---|
| Q1 | What share of configured entitlement values historically came from the documents vs verbal customer input? This sets the ceiling on automation. | Data / SMEs | `yes` |
| Q2 | Can historical customer documents be used for evaluation under existing contracts, and with what anonymisation? | Legal / Privacy | `yes` |
| Q3 | Is the canonical unit hours everywhere, or should day-based entitlements stay in days until configuration? | Product / SMEs | no |
| Q4 | Should statutory minimums be shown as advisory context (not extracted values) in v1? | Legal / Product | no |
| Q5 | Do Canadian customers require in-country inference, and does our provider support it for the chosen models? | Security / Eng | `yes for CA beta` |
| Q6 | Who owns rule packs long-term: Services Methodology or the product team? | Director | no |
| Q7 | Does hiding the value on low-band fields actually reduce anchoring, or just slow review? A/B in alpha. | Design / Research | no |

## 19. Appendix

### A. Worked example

**Source (Employee Handbook 2025, p. 31, §6.2):**

> “Regular full-time employees become eligible for Paid Time Off after completing ninety (90) days of continuous service. PTO accrues at 1.54 hours per bi-weekly pay period. Employees may not accrue more than 120 hours; once the maximum is reached, no further PTO will accrue until the balance falls below the cap.”

The handbook says nothing about carryover or termination. *PTO Policy Addendum (2023), p. 2* says "up to 40 hours may be carried into the next calendar year."

```json
{
  "entitlement_type": { "status": "extracted", "value": "VACATION_PTO",
    "verbatim": "Paid Time Off", "citations": [{"doc":"handbook-2025","page":31,"span":[88,101]}] },
  "policy_bundle":    { "status": "extracted", "value": "Regular full-time",
    "verbatim": "Regular full-time employees", "citations": [{"doc":"handbook-2025","page":31,"span":[0,27]}] },
  "eligibility.waiting_period": { "status": "extracted", "value": {"amount":90,"unit":"calendar_days"},
    "verbatim": "ninety (90) days of continuous service", "confidence": 0.97, "citations": [...] },
  "accrual.rate":     { "status": "extracted", "value": {"amount":1.54,"unit":"hours","per":"biweekly_pay_period"},
    "verbatim": "1.54 hours per bi-weekly pay period", "confidence": 0.98, "citations": [...] },
  "balance.cap":      { "status": "extracted", "value": {"amount":120,"unit":"hours"},
    "verbatim": "may not accrue more than 120 hours", "confidence": 0.95, "citations": [...] },
  "carryover":        { "status": "extracted", "value": {"amount":40,"unit":"hours"},
    "verbatim": "up to 40 hours may be carried into the next calendar year",
    "citations": [{"doc":"pto-addendum-2023","page":2,"span":[...]}] },
  "year_basis":       { "status": "extracted", "value": "calendar", "confidence": 0.71,
    "verbatim": "next calendar year", "note": "inferred from carryover clause only" },
  "termination_payout": { "status": "not_stated", "value": null },
  "gaps": [
    { "field": "termination_payout", "severity": "required",
      "question": "Your handbook (p.31) and PTO addendum don't say what happens to unused PTO when an employee leaves. Is it paid out, forfeited, or handled per state law?" }
  ]
}
```

Note that `year_basis` lands in the Low band. It is supported by text, but only indirectly, so the consultant must confirm it. The addendum is older than the handbook, so the review UI also asks the consultant to confirm the addendum is still in force.

### B. Gap & conflict taxonomy

| Code | Meaning | Example |
|---|---|---|
| GAP-REQ | Required field not stated anywhere | Termination payout missing |
| GAP-PARTIAL | Rule stated incompletely | Tenure tiers listed for 0–5 years only |
| GAP-UNIT | Unit can't be normalised from stated facts | "10 days" with no hours-per-day |
| GAP-POP | Population unclear | "Employees" with part-time rules elsewhere |
| CONF-VAL | Two sources, different values | Handbook 10 days vs policy 12 days |
| CONF-VER | Possibly superseded document | 2022 handbook alongside 2025 |
| REF-LAW | Defers to statute | "as required by applicable law" |

### C. Glossary

| Term | Definition |
|---|---|
| Entitlement | A right to paid or unpaid time away from work, tracked as a balance that accrues and is drawn down |
| Accrual | How balance is earned: per pay period, per hour worked, or front-loaded annually |
| Policy bundle | The group of employees a set of entitlement rules applies to |
| Carryover | Unused balance allowed into the next entitlement year |
| Citation / span | Document, version, page and character range (plus bounding box) that supports a value |
| Gap | A required rule the customer's documents do not state |
| Rule pack | Versioned SME configuration: schema, lexicon, required fields, instructions per entitlement type × jurisdiction |

---

*Entitlement Extraction Assistant · PRD v0.4 draft · Practice document for system-design preparation. Next artefacts: high-level design, then a low-level design per pipeline stage.*
