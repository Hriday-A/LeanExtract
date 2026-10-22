# Customer Brief: Receipt Extraction for Meridian Foodservice Group

*Fictional client, written for the LeanExtract portfolio project. Version 1.0, 2026-10-22.*

## Client and problem

Meridian Foodservice Group (fictional) is a mid-size distributor with about 2,000 employees. It operates in Indonesia and is expanding into the United States. Its **Travel & Expense team** has 3 people who key data in by hand. They process about **25,000 receipts a month** from employee expense claims: they type the line items, subtotal, tax and total into the ERP, then check each claim against policy.

The team wants each receipt image to become structured JSON automatically. People should only handle the receipts the system is unsure about.

## Constraints

| # | Constraint | Detail |
|---|---|---|
| C1 | **Data residency** | Receipts must not leave Meridian's network by default. That is why the primary model has to be a small model they host themselves. |
| C2 | **Approved exception path** | Security has approved one API model, used under Meridian's enterprise agreement with zero data retention, **only as a fallback** and capped at **10% of monthly volume**. Every document sent to it is logged. |
| C3 | **Hardware** | One on-prem server with a single 16 GB NVIDIA GPU (T4-class). There is no budget for more GPUs in year one. |
| C4 | **Cost ceiling** | **≤ $1.00 per 1,000 documents**, all-in. That includes local GPU compute, priced at a documented public on-demand T4 hourly rate, plus any fallback API spend. |
| C5 | **Latency** | The single-document endpoint, which employees use when they upload in the expense app, must have **p95 ≤ 5 s** for extraction on the T4 (OCR is reported separately). Batch processing must handle the nightly backlog: **≥ 2,000 documents/hour**. |
| C6 | **Retrainable in-house** | Meridian's analysts must be able to retrain on their own labeled receipts from a runbook, without ML specialists. |

## Output schema (provisional)

```json
{
  "store_name": "string | null",
  "date": "YYYY-MM-DD | null",
  "line_items": [{"name": "string", "quantity": "number", "price": "number"}],
  "subtotal": "number | null",
  "tax": "number | null",
  "total": "number"
}
```

`store_name` and `date` stay in the schema **only if CORD actually labels them**. This gets checked in Phase 1. If they are dropped, every criterion below that mentions them is dropped too, and the change is recorded in the change log.

## Acceptance criteria

All criteria are scored on the **CORD official test split**, under both input conditions: **clean** text rebuilt from the annotations, and **OCR** text from a real OCR engine. The **OCR condition is the one that decides pass/fail**, because that is what production input looks like. The model under test is the fine-tuned model plus the full production pipeline (validation and fallback), unless a row says otherwise.

| ID | Metric | Threshold (OCR condition) | Why this threshold |
|---|---|---|---|
| AC1 | Total-amount accuracy (numeric match after normalization) | **≥ 90%** (clean: ≥ 95%) | A wrong total means a wrong reimbursement. This is the field the business can least afford to get wrong. |
| AC2 | Field-level exact match, header fields (subtotal, tax, total, plus store/date if kept) | **≥ 85%** | Each wrong header field is a manual correction. Below 85%, reviewing the output costs about as much as keying it in. |
| AC3 | Line-item F1 | **≥ 0.80** | Line items feed policy checks, such as alcohol, but they are not paid out directly, so they can tolerate more error than the total. |
| AC4 | Valid-JSON rate | **≥ 99%** for the raw fine-tuned model; **100%** after fallback | Invalid JSON breaks the ERP import, so the pipeline must never pass it downstream. |
| AC5 | Parity with the API model (fine-tuned model alone, no fallback) | Within **3 percentage points** of the API model on AC1 and AC2, or better | This is the core claim: a small local model is good enough. |
| AC6 | Blended cost | **≤ $1.00 / 1,000 docs** (C4); ratio to API-only cost reported | Keeps the project within budget. The ratio is reported whichever way it comes out. |
| AC7 | Latency | Single-doc **p95 ≤ 5 s**; batch **≥ 2,000 docs/hour** (C5) | Employees will wait about 5 seconds in the app. The nightly batch must finish before morning. |
| AC8 | Fallback rate | **≤ 10%** of documents routed to the API model | This is the governance cap from C2. Above it, residency is effectively broken. |
| AC9 | Human-review queue | Flags **≤ 15%** of documents and catches **≥ 70%** of the documents whose total is wrong | Three reviewers can handle about 15% of volume. The queue only has value if it catches most real errors. |
| AC10 | U.S. receipts, held-out set | Report the drop before adaptation. After adaptation, total-amount accuracy **≥ 85%** and **at least half** of the drop recovered | This is the U.S. expansion. CORD is Indonesian, so some drop is expected, and we need proof that retraining fixes it. |
| AC11 | Privacy | **Zero** card numbers, personal names or personal addresses in the repo, the Hub or the Space | This is a hard gate. Failing it fails the project, whatever the accuracy. |

## Out of scope

Handwritten receipts, multi-page invoices, currency conversion, training an OCR engine, and checking claims against expense policy (that happens downstream).

## Scoring rules

1. These criteria are **frozen when this brief is committed**. Any later change goes in the change log below with a reason, and the final scoring in Phase 9 is reported against the **original** thresholds as well.
2. Every reported number must come from a logged run: a W&B run ID or a committed results file.
3. A criterion that is missed is reported as missed, with the gap and a proposed remedy.

## Change log

| Date | Change | Reason |
|---|---|---|
| 2026-10-22 | v1.0 created | — |
