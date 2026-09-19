# Chunk 01 · Ingest, normalise & segment

> **The one-line claim:**
> *Every citation the system will ever show is only as good as the coordinates this stage records. Ingest turns a messy pile of files into one canonical, ordered, addressable set of clauses — and it never loses a page silently.*

| | |
|---|---|
| **Chunk** | 01 of 07 |
| **PRD requirements** | FR-ING-01 → FR-ING-05 |
| **Upstream** | Consultant upload (review workspace) |
| **Downstream** | Chunk 02 — Relevance screen & routing |
| **Status** | Deep dive ✅ · HLD diagram ⏳ · LLD ⬜ · Code ⬜ |

---

## 1. Why this chunk exists

Listen to the audio summary for this section:

<audio controls preload="metadata">
  <source src="voiceover/01-why-this-chunk-exists.mp3" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>

The rest of the system makes one promise: **every value points to the exact sentence it came from.** That promise is kept or broken here, before any model is involved.

If ingest drops a page, the extractor can't cite it. If ingest records the wrong position for a sentence, the reviewer sees the wrong highlight — and **a wrong highlight is worse than no highlight**, because it looks like evidence. If ingest splits a clause in half, the extractor sees "accrues at 1.54 hours" in one piece and "per bi-weekly pay period" in another.

So this chunk has a narrow job and a high bar:

1. Accept whatever the customer sent.
2. Lose nothing.
3. Produce **one canonical shape** — ordered pages, text, images, and clause-level segments with exact coordinates — that every later stage consumes without caring whether the source was a Word file or a fax.

---

## 2. Scope

| This chunk owns | This chunk does **not** own |
|---|---|
| File intake, validation, de-duplication | Deciding what is relevant (chunk 02) |
| Page rendering and text extraction, including OCR | Interpreting any rule (chunk 03) |
| Structure detection: headings, clauses, lists, tables | Entitlement vocabulary or synonyms (chunk 06) |
| Segmentation with page, character offsets and bounding box | Human review UI (chunk 05) |
| The fidelity invariants (nothing lost, nothing moved) | Quality scoring against gold (chunk 04) |
| Storing originals immutably | Scaling, queues, retries at platform level (chunk 07) |

---

## 3. The domain: what a policy document actually looks like

A typical input set for **one** implementation:

| Document | Format | Pages | What's in it |
|---|---|---|---|
| Employee Handbook 2025 | DOCX | 84 | Welcome letter, values, conduct, **§6 Time Off**, benefits, IT policy |
| PTO Policy Addendum 2023 | PDF (born digital) | 3 | Carryover rule, request process |
| Collective Agreement — Warehouse Local 412 | Scanned PDF | 112 | Articles, **Article 18 Vacation** with a tenure table |
| Ontario Addendum | PDF | 6 | Provincial sick-day wording |
| Offer letter template | DOCX | 2 | "You will receive 15 days of vacation…" |
| Employee Handbook 2022 | PDF | 79 | Superseded, but the customer sent it anyway |
| …eight more | mixed | ~300 | Codes of conduct, travel, expenses, dress code |

**~600 pages. Perhaps 15–40 clauses that matter.**

The anatomy that makes this hard:

- **Rules hide in tables.** "0–4 years: 10 days · 5–9 years: 15 days · 10+ years: 20 days." Flatten that table into a sentence stream and the tiers become ambiguous.
- **Clauses are numbered and nested.** `6.2 (b)(ii)` is an exception to `6.2 (b)`. Lose the hierarchy and you lose the scope of the exception.
- **Headers and footers repeat on every page.** "Confidential — ACME Corp — Rev 3" appears 84 times. Left in, it pollutes every segment and every relevance decision.
- **Cross-references.** "Accrual rates are shown in Appendix B." The rule lives 60 pages away from the sentence that introduces it.
- **Word files have no pages.** A DOCX is a flow of paragraphs; "page 31" only exists once it's rendered. But reviewers and customers talk in pages.
- **Scans have no text.** The collective agreement is a photograph of paper. Some pages are skewed, some are stamped, one is upside down.
- **Two languages.** A Quebec addendum may be bilingual, side by side.
- **Tracked changes and comments.** HR drafts sometimes arrive with redlines still in them. Which text is the policy — the deletion or the insertion?

---

## 4. The contract — what this chunk produces

Everything downstream sees **this shape and nothing else.**

### 4.1 Document

| Field | Meaning |
|---|---|
| `doc_id` | Stable id for this document within the project |
| `content_hash` | SHA-256 of original bytes — dedup key and audit anchor |
| `source_filename`, `mime_type`, `uploaded_by`, `uploaded_at` | Provenance |
| `page_count_declared` | Pages the source claims to have (PDF page tree, TIFF frames, rendered DOCX) |
| `page_count_produced` | Pages this stage actually produced — **must equal the above** |
| `status` | `ready` · `ready_with_warnings` · `failed` · `superseded` |
| `warnings[]` | e.g. `low_ocr_confidence_pages`, `tracked_changes_present`, `encrypted` |
| `language[]` | Detected per page, rolled up |

### 4.2 Page

| Field | Meaning |
|---|---|
| `page_no` | 1-based, as a human would say it |
| `image_ref` | Rendered page image (fixed DPI) in object storage |
| `text_source` | `native` (text layer / DOCX) or `ocr` |
| `ocr_confidence` | Mean word confidence, when OCR was used |
| `width`, `height` | Coordinate space for every bounding box on this page |

### 4.3 Segment — the unit everything else works on

| Field | Meaning |
|---|---|
| `segment_id` | Stable, deterministic: `doc_id:page:ordinal` |
| `type` | `heading` · `clause` · `list_item` · `table` · `table_row` · `footnote` |
| `text` | Clean text of the segment |
| `section_path` | e.g. `["6 Time Off", "6.2 Paid Time Off", "(b)"]` — the heading breadcrumb |
| `anchors[]` | One per page the segment touches: `page_no`, `char_start`, `char_end`, `bbox[]` |
| `table` | For tables: rows × columns with cell text and cell boxes |
| `refs[]` | Detected cross-references ("see Appendix B") — resolved later in chunk 02 |
| `is_boilerplate` | Repeated header/footer text, kept but marked |

### 4.4 Worked example

Source — Employee Handbook 2025, page 31:

> **6.2 Paid Time Off**
> Regular full-time employees become eligible for Paid Time Off after completing ninety (90) days of continuous service. PTO accrues at 1.54 hours per bi-weekly pay period.
> Employees may not accrue more than 120 hours; once the maximum is reached, no further PTO will accrue until the balance falls below the cap.
> *Confidential — ACME Corp — Rev 3 · Page 31*

Produced:

```json
[
  { "segment_id": "hb25:31:004", "type": "heading",
    "text": "6.2 Paid Time Off",
    "section_path": ["6 Time Off"],
    "anchors": [{ "page_no": 31, "char_start": 812, "char_end": 829, "bbox": [72, 140, 260, 158] }] },

  { "segment_id": "hb25:31:005", "type": "clause",
    "text": "Regular full-time employees become eligible for Paid Time Off after completing ninety (90) days of continuous service. PTO accrues at 1.54 hours per bi-weekly pay period.",
    "section_path": ["6 Time Off", "6.2 Paid Time Off"],
    "anchors": [{ "page_no": 31, "char_start": 830, "char_end": 1003, "bbox": [72, 162, 540, 210] }] },

  { "segment_id": "hb25:31:006", "type": "clause",
    "text": "Employees may not accrue more than 120 hours; once the maximum is reached, no further PTO will accrue until the balance falls below the cap.",
    "section_path": ["6 Time Off", "6.2 Paid Time Off"],
    "anchors": [{ "page_no": 31, "char_start": 1004, "char_end": 1144, "bbox": [72, 214, 540, 248] }] },

  { "segment_id": "hb25:31:007", "type": "footnote", "is_boilerplate": true,
    "text": "Confidential — ACME Corp — Rev 3 · Page 31", "...": "..." }
]
```

When chunk 03 later says *"cap = 120 hours, cited from `hb25:31:006`, chars 1004–1144"*, the review screen can draw a box around exactly that sentence on exactly that page. **That capability is created here.**

---

## 5. Numbered flow

> The HLD workflow diagram for this chunk is the next step and will be added as `diagrams/mermaid/01-ingest-flow.mmd`.

| # | Step | What happens | Why it's shaped this way |
|---|---|---|---|
| **1** | **Intake** | Consultant uploads the set into a project. Each file gets a `content_hash`. Identical bytes already in the project are linked, not reprocessed. | Customers resend the same file. Re-processing is wasted cost and creates duplicate citations. |
| **2** | **Pre-flight** | Check type (by magic bytes, not extension), size, page count, encryption, corruption. Reject early with the file name and the reason. | Fail before paying for any conversion. An extension lies; magic bytes don't. |
| **3** | **Type-keyed dispatch** | A registered loader per format: PDF, scanned PDF, DOCX, DOC, TIFF, image. | A new format is a registration, not an edit to a chain of `if` statements. |
| **4** | **Render to pages** | Every document becomes an ordered list of page images at fixed DPI. **DOCX/DOC are rendered to PDF first** so "page 31" means the same thing to the system, the consultant and the customer. Multi-frame TIFFs are unrolled frame by frame. | Citations need a stable page and coordinate space. A Word file has neither until it's rendered. |
| **5** | **Text per page** | Use the native text layer where it exists and is healthy. Otherwise OCR. Record `text_source` and confidence per page. | Born-digital text is exact and free. OCR is the fallback, and its confidence must travel with the text. |
| **6** | **Fidelity check #1 — pages in = pages out** | `page_count_declared == page_count_produced`, per document. Mismatch → document fails loudly. | The failures that hurt are the silent ones. A missing page produces no error — only a missing rule. |
| **7** | **Layout & structure** | Detect reading order (including two-column layouts), headings and numbering, list items, tables with cell structure, and repeated headers/footers. | The meaning of a clause depends on its section and its parent clause. Tables are frequently *the* rule. |
| **8** | **Segment** | Split into clause-level segments along the detected structure — never at a fixed token count. A clause spanning a page break becomes one segment with two anchors. Tables stay whole. | A fixed-size chunk can cut "1.54 hours" from "per bi-weekly pay period". Structure-aware splitting keeps each rule intact. |
| **9** | **Anchor** | Every segment records page, character offsets into that page's text, and bounding boxes. | This is the raw material of every citation downstream. |
| **10** | **Fidelity check #2 — anchors round-trip** | For every segment, slicing the page text at `char_start:char_end` must reproduce the segment text exactly. | If this ever fails, a highlight will land on the wrong words. Catch it here, not in front of a consultant. |
| **11** | **Flag, don't hide** | Low OCR confidence, tracked changes, empty pages, mixed languages → document or page warnings, visible in the review workspace. | A warning the consultant can see beats a guess the consultant can't. |
| **12** | **Persist & hand off** | Originals (immutable), page images, page text, segments and a manifest are stored, keyed by hash. The document is marked `ready` and chunk 02 is notified. | Idempotent: rerunning with the same bytes and the same pipeline version costs nothing. |

---

## 6. How it evolved

### Iteration 0 — send each file straight to a model

Upload the PDF to a large-context model and ask for the entitlements.

**Why it fails:**
- No coordinates, so no citations. The model can quote text, but you can't draw a box around it or prove where it came from.
- Word files, scans and multi-frame TIFFs behave differently per provider, and some drop pages silently.
- You pay to send 600 pages of welcome letters and dress codes to the most expensive model in the pipeline.
- Nothing is reusable. Every change to the prompt re-sends everything.

### Iteration 1 — extract text, split into fixed-size chunks

The classic RAG pipeline: text extraction, 500-token chunks with overlap, embed, retrieve.

**Four ways it broke for this domain:**
1. **Chunks cut rules in half.** The accrual rate lands in one chunk and its unit in the next. The overlap helps sometimes and duplicates text other times.
2. **Tables flattened into word soup.** The tenure table becomes "0 4 years 10 days 5 9 years 15 days", and the tiers are no longer reliably readable.
3. **Headers and footers everywhere.** "Confidential — ACME Corp" becomes the most frequent phrase in the corpus.
4. **Chunk offsets aren't page coordinates.** You can say "chunk 212", but a consultant can't open chunk 212. You can't highlight it.

### Iteration 2 — the design above

| Failure | Fix |
|---|---|
| No coordinates | Render every page; anchor every segment with page, offsets and bbox |
| Rules cut in half | Structure-aware, clause-level segmentation — never fixed-size |
| Tables destroyed | Tables detected and kept as structured segments with cell boxes |
| Boilerplate noise | Repeated headers/footers detected across pages and marked |
| Word files have no pages | Render DOCX to PDF, then treat it like any other PDF |
| Silent page loss | Pages-in = pages-out asserted per document |
| Wrong highlights | Anchor round-trip asserted per segment |

---

## 7. Key design decisions

### D1 · Clause-level segments, not fixed-size chunks
- **Bought:** each rule stays intact; citations point to a meaningful unit; segments carry their section path.
- **Cost:** layout analysis is harder than splitting on tokens and can fail on badly formatted documents.
- **Fallback:** if structure detection fails on a page, fall back to paragraph-level segments for that page and flag it. Never fall back to fixed-size chunks.

### D2 · Render Word files to PDF before anything else
- **Bought:** one coordinate system for every format; "page 31" means the same to system, consultant and customer; the same review UI works for everything.
- **Cost:** a rendering dependency (a headless office converter). Pagination can differ slightly from the customer's own Word view.
- **Mitigation:** keep the DOCX paragraph id alongside each anchor, so a citation can also be expressed as "§6.2, paragraph 3" when pagination differs.

### D3 · Keep page images for every page
- **Bought:** bounding-box highlights, which turn review from *re-reading* into *confirming*. That is where the consultant's hours are saved.
- **Cost:** storage. Around 600 pages × ~300 KB ≈ **~180 MB per implementation** for this domain. Estimate — confirm with real renders. Negligible against the value.

### D4 · Mark boilerplate, don't delete it
- **Bought:** downstream stages ignore it by default; nothing is lost if detection is wrong; round-trip anchors still hold.
- **Cost:** slightly more data carried forward.

### D5 · Tracked changes: render the accepted state, warn loudly
- A redlined DOCX is ambiguous: is the policy the old text or the new?
- **Decision:** process the document as if all changes were accepted, attach warning `tracked_changes_present`, and surface it at the top of the review workspace so the consultant confirms with the customer.
- **Alternative rejected:** silently rejecting the file. It stalls the project over something a consultant can resolve in one email.

### D6 · Native text first, OCR as fallback — per page, not per document
- A "scanned" PDF often has a few born-digital pages (a typed cover sheet), and a "digital" PDF often has one scanned signature page.
- **Decision:** choose native vs OCR per page based on text-layer health (character count, garbage-character ratio). Record which was used.

---

## 8. Invariants

These are checks in code, not intentions. Each one fails loudly.

| # | Invariant | If violated |
|---|---|---|
| **I1** | Pages declared = pages produced, per document | Document `failed`; reason shown to consultant |
| **I2** | Every segment's anchors round-trip to its exact text | Document `failed`; engineering alert |
| **I3** | Every page has either text or an explicit `empty` / `unreadable` flag | Page flagged; never passed as silently empty |
| **I4** | `segment_id` is deterministic for the same bytes + pipeline version | Build fails in CI |
| **I5** | Originals are never modified after intake | Write-once storage policy |
| **I6** | Every run records the pipeline version, loader versions and OCR engine version | Run rejected |

---

## 9. Failure modes & graceful degradation

| Failure | Behaviour |
|---|---|
| Unsupported or disguised type | Rejected at pre-flight with file name and reason. Never guessed at with the nearest loader. |
| Password-protected file | Flagged `encrypted`; consultant asked to request an unlocked copy. Never silently skipped. |
| Corrupt file | That document fails with its error; the rest of the batch continues. Re-runnable on its own. |
| Page-count mismatch | Document fails loudly. **Partial conversion reported as complete is the failure this chunk exists to prevent.** |
| Low OCR confidence on a page | Page processed, flagged `low_ocr_confidence`; review UI tells the consultant to read that page themselves. |
| Structure detection fails on a page | Falls back to paragraph segments for that page, flagged. |
| Blank page | Marked `empty`. Distinguished from `unreadable` — a scan that produced no text is not a blank page. |
| Duplicate file | Linked to the existing document by hash; no reprocessing. |
| Superseded version (P1) | Likely-version pairs detected by title and similarity; consultant chooses which is current. Superseded doc kept for audit, excluded from extraction. |
| Tracked changes | Accepted state processed; `tracked_changes_present` warning shown prominently. |

---

## 10. What we measure at this stage

This stage has no model accuracy — it has **fidelity**.

| Metric | Target | Why |
|---|---|---|
| Page conservation failures | 0 unexplained | The core invariant |
| Anchor round-trip failures | 0 | A wrong highlight is worse than none |
| Pages on OCR fallback | tracked | A sudden rise means the customer mix changed |
| Pages flagged low-confidence | tracked, alert on spike | Predicts downstream accuracy loss |
| Table detection recall on golden set | ≥ 95% | Tenure tables are often the rule itself |
| Median ingest time per 600-page set | ≤ 10 min | Leaves room in the 30-min end-to-end target (PRD NFR) |

---

## 11. Back of the envelope

Estimates, to be confirmed with real documents.

| Quantity | Estimate |
|---|---|
| Pages per implementation, entitlements | ~600 |
| Share of pages needing OCR | ~20–30% |
| Render + native text, per page | ~0.2 s |
| OCR, per page | ~1–2 s |
| Serial time for 600 pages | ~5–8 min |
| With 8 workers in parallel | < 1–2 min |
| Storage per implementation | ~180 MB images + a few MB text |
| Across 10–12 domains per customer | ~7,000 pages, ~2 GB |

**The conclusion that matters:** ingest is not the bottleneck. It's CPU-bound, parallel per page, and cheap. The bottleneck of the whole system is model throughput in chunks 02–03, which is why this stage should do everything it can deterministically, and hand the models as little as possible.

---

## 12. PRD traceability

| PRD requirement | Covered by |
|---|---|
| FR-ING-01 · Accept PDF, scanned PDF, DOCX, DOC, TIFF/PNG up to 200 files / 2,000 pages | Steps 1–3 |
| FR-ING-02 · De-duplicate by hash; detect versions (P1) | Step 1, §9 superseded |
| FR-ING-03 · Render pages, text layer or OCR, pages in = pages out, OCR flags | Steps 4–6, I1, I3 |
| FR-ING-04 · Segment with structure, page, offsets, bbox; tables keep structure | Steps 7–10, I2 |
| FR-ING-05 · Originals immutable; derived artefacts reference original by hash | Step 12, I5 |

---

## 13. Questions this chunk will be asked

**"Why not just send the PDF to the model?"**
Because I need coordinates, not just text. The product promise is a highlight on the exact sentence. A model can quote, but it can't give me a page-anchored bounding box I can verify. And I don't want to pay the most expensive model to read 600 pages of dress codes.

**"Why not fixed-size chunks like normal RAG?"**
Because the unit of meaning here is a clause, not 500 tokens. Fixed chunks cut "1.54 hours" from "per bi-weekly pay period" and turn tenure tables into word soup. I split along the document's own structure and keep tables whole.

**"Word files don't have pages. How do you cite them?"**
I render them to PDF first, so every format shares one coordinate system and "page 31" means the same thing to everyone. I also keep the Word paragraph id, so the citation still holds if pagination differs slightly on the customer's machine.

**"What's the most expensive mistake in this stage?"**
Losing a page silently. Nothing errors — the extractor just never sees the rule, and chunk 03 correctly reports it as "not stated". We'd ask the customer a question their own policy already answers. So pages-in equals pages-out is asserted per document.

**"How do you know a highlight lands on the right words?"**
Every segment's anchor must round-trip: slice the page text at its offsets and you get exactly the segment text. If not, the document fails before any consultant sees it.

**"What do you do with a bad scan?"**
Process it, but flag the page with its OCR confidence. The review screen tells the consultant to read that page themselves. I'd rather show uncertainty than hide it.

**"Handbook has tracked changes. Which text do you use?"**
The accepted state, with a loud warning at the top of the review. It's a question for the customer, not a decision for the pipeline.

**"How do you add a new file type?"**
Register a loader. Dispatch is keyed by detected type, so nothing existing is touched.

---


## 14. Open questions

| # | Question | Blocks |
|---|---|---|
| Q1 | Which OCR engine: cloud document-AI service vs self-hosted? Driven by data-residency needs for Canadian customers. | LLD |
| Q2 | Which layout/table detector gives the best table recall on real collective agreements? Needs a bake-off on ~20 documents. | LLD |
| Q3 | Render DPI: 150 vs 200 vs 300 — OCR accuracy vs storage. | LLD |
| Q4 | How often do DOCX paginations differ between our renderer and the customer's Word? Does it confuse consultants? | Alpha |
| Q5 | Bilingual side-by-side pages: segment per language column, or keep paired? | LLD (P1, French-Canadian) |
| Q6 | Do we ever need to process embedded attachments (a PDF inside a DOCX)? | Discovery |

---

*Next: HLD workflow diagram for this chunk → `diagrams/mermaid/01-ingest-flow.mmd`, then redrawn in Excalidraw.*
