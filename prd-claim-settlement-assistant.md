# PRD: Claim Settlement Assistant

## 1. Introduction / Overview

AR and Finance analysts today manually review deductions within Oracle Channel Revenue Management — a time-consuming, error-prone process that delays cash recovery and increases operational cost. The **Claim Settlement Assistant** is an AI-powered digital coworker that automates and accelerates the resolution of these deductions.

The assistant operates in two modes:
- **Autonomous resolution** — for high-confidence cases where the claim contains enough information to settle without human intervention.
- **Recommended resolution** — for lower-confidence cases where the assistant surfaces a suggested action for an analyst to review and confirm.

The MVP focuses exclusively on deductions as the claim source. Other claim sources (e.g., overpayments) are planned for future releases.

---

## 2. Goals

1. Autonomously gather resolution clues from structured and flexible claim fields (customer reason, customer reference, receipt application descriptive flexfields).
2. Automatically resolve promotional deductions by exact-matching program accruals and fully settling the claim (with configurable pay-over support per customer).
3. Automatically resolve non-promotional deductions by exact-matching and applying one or more open credit memos (with configurable customer tolerance).
4. Autonomously classify each claim's type and reason using a RAG (Retrieval-Augmented Generation) knowledge document.
5. Log every resolution attempt — successful or not — with a full record of what was found and the reasoning behind the outcome.
6. Reduce the time AR/Finance analysts spend manually processing deductions.
7. Decrease end-to-end settlement cycle time (from deduction creation to resolution).
8. Increase the percentage of deductions resolved without human intervention.

---

## 3. User Stories

**US-1 — Clue Gathering**
> As the Claim Settlement Assistant, I need to autonomously gather clues for how to resolve a claim from the customer reason, customer reference, or a receipt application descriptive flexfield, so that I have the right inputs to attempt an autonomous resolution before involving an analyst.

**US-2 — Promotional Deduction Auto-Resolution**
> As the Claim Settlement Assistant, I need to autonomously resolve a promotional deduction by associating available program accruals if a program or program code has been provided in the claim. The match must be exact. There must be enough accrual balance to fully settle the claim, unless pay-over is enabled for the customer and the shortfall is within the configured pay-over threshold.

**US-3 — Non-Promotional Deduction Auto-Resolution (Credit Memo)**
> As the Claim Settlement Assistant, I need to autonomously resolve a non-promotional deduction by applying one or more open credit memos if credit memo references have been provided in the claim. The match must be exact. The combined credit memo balance must fully cover the claim, or the shortfall must be within the customer's configured tolerance level.

**US-4 — Claim Classification**
> As the Claim Settlement Assistant, I need to autonomously classify the claim type and claim reason correctly, using a RAG document as my knowledge source, so that each deduction is routed or resolved with the correct context.

**US-5 — Resolution Logging**
> As the Claim Settlement Assistant, I need to log how I resolved each claim and based on what information I found. If a claim was not resolved, I must still log what was found and the specific reason why resolution could not be completed.

**US-6 — Analyst Visibility**
> As an AR analyst, I want to see a clear, plain-language log of what the assistant did (or attempted) on each claim, so that I can trust the outcome, take over where needed, and satisfy audit requirements.

---

## 4. Functional Requirements

### 4.1 Clue Gathering

1. The system must inspect the following fields on each incoming claim to identify resolution clues:
   - **Customer Reason** — free-text or coded reason provided by the customer.
   - **Customer Reference** — reference number or identifier provided by the customer (e.g., a program code, credit memo number, or PO number).
   - **Receipt Application Descriptive Flexfields** — all descriptive flexfield segments attached to the receipt application record must be inspected. The set of descriptive flexfield segments is consistent across all customers, so no customer-specific configuration is required for this step.
2. The system must extract any program names, program codes, or credit memo numbers found across all inspected fields and treat them as candidate inputs for resolution.
3. If no usable clues are found in any of these fields, the system must log this outcome and skip autonomous resolution, routing the claim for analyst review.

### 4.2 Claim Classification

4. The system must classify each claim with a **claim type** and **claim reason** before attempting resolution.
5. Classification must be performed using a **RAG (Retrieval-Augmented Generation) approach**: the assistant retrieves relevant rules or examples from a configured knowledge document and uses them to determine the correct claim type and reason.
6. The RAG classification knowledge document is owned and maintained by the **Claim Analyst Supervisor**. It must be updatable without a code change. It is expected to be revised approximately twice a year.
7. The classified claim type and reason must be written back to the claim record in Oracle Channel Revenue Management only if the RAG confidence level meets or exceeds the configured confidence threshold.
8. The confidence threshold must default to **90%** system-wide but must be **configurable per customer, per settlement method, and per RAG use case** via a dedicated **confidence threshold RAG document**. This document supports entries at any combination of those three dimensions and must be updatable without a code change.
9. If the confidence level is below the applicable threshold, the system must not write back a classification and must instead flag the claim for analyst review, logging the actual confidence score and the threshold that was not met.

### 4.3 Resolution Path 1 — Promotional Program Resolution

9. The system must attempt promotional resolution when a program name or program code is identified in the clue-gathering step.
10. The system must query Oracle Channel Revenue Management for programs whose name or code is an **exact match** to the value(s) found in the claim. Fuzzy or partial matching is not permitted.
11. **Multiple programs found — error condition:** If clues from the inspected fields resolve to more than one distinct program, the system must **not** auto-resolve. It must log this as an error (e.g., "Multiple programs identified — cannot determine correct program for resolution"), and route the claim for analyst review.
12. If exactly one program is matched, the system must retrieve all available accrual line balances for that program.
13. **Multi-accrual combining:** The system must combine multiple accrual lines from the matched program as needed to cover the claim amount. Accruals must be associated in **FIFO order based on the requested accounting date (earned date)** — the earliest-earned accrual lines are applied first.
14. **Full settlement (standard):** If the combined accrual balance across all available lines is greater than or equal to the claim amount, the system must autonomously associate the accrual lines to the claim and mark the deduction as resolved.
15. **Pay-over settlement (optional, per customer):** If the combined accrual balance is less than the claim amount but pay-over is enabled for the customer, the system must check whether the shortfall is within the customer's configured pay-over threshold. If it is, the system must proceed with settlement using all available accruals.
16. **Partial resolution — write-off:** If the combined accrual balance is insufficient and pay-over is not applicable, the system must consult the **customer tolerance RAG document** to determine the customer's tolerance amount. If the remaining unsettled claim amount is less than the customer's tolerance, the system must resolve the remaining balance as an automatic **write-off** and mark the claim as fully resolved.
17. **Partial resolution — error:** If the remaining unsettled amount meets or exceeds the customer's tolerance, the system must **not** auto-resolve. It must log this as an error, record the shortfall amount and tolerance value, and route the claim for analyst review.
18. If no exact program match is found, the system must **not** auto-resolve. It must log the finding and route the claim for analyst review.

### 4.4 Resolution Path 2 — Non-Promotional Credit Memo Resolution

16. The system must attempt credit memo resolution when one or more credit memo references are identified in the clue-gathering step.
17. The system must query Oracle Fusion ERP Receivables to locate each referenced credit memo. The match between the reference and the credit memo number must be an **exact match**. Fuzzy or partial matching is not permitted.
18. The system must validate each matched credit memo:
    - It must exist in Oracle Fusion ERP.
    - It must have an open (unapplied) balance.
    - It must belong to the same customer as the deduction.
    - Its currency must exactly match the currency of the deduction. Cross-currency credit memo application is not supported.
19. **Full settlement (standard):** If the combined open balance of all exact-matched credit memos is greater than or equal to the claim amount, the system must autonomously apply the credit memo(s) to the deduction and mark it as resolved.
20. **Tolerance-based settlement — write-off:** If the combined open balance is less than the claim amount, the system must consult the **customer tolerance RAG document** to determine the customer's tolerance amount. If the remaining unsettled claim amount is less than the customer's tolerance, the system must resolve the remaining balance as an automatic **write-off** and mark the claim as fully resolved.
21. **Partial resolution — error:** If the remaining unsettled amount meets or exceeds the customer's tolerance, the system must **not** auto-resolve. It must log this as an error, record the shortfall amount and tolerance value, and route the claim for analyst review.
22. If no exact credit memo match is found, or all matched credit memos are already fully applied, belong to a different customer, or have a currency mismatch, the system must **not** auto-resolve. It must log the specific validation failure(s) and route the claim for analyst review.

### 4.5 Resolution Logging

23. The system must create a log entry for **every claim it processes**, regardless of whether resolution was successful.
24. Each log entry must include:
    - Claim identifier and claim source type.
    - Clues found (fields inspected, values extracted).
    - Classified claim type and claim reason (and confidence level if available).
    - Resolution path attempted (promotional, credit memo, or none).
    - Specific records evaluated (program name/code, accrual lines and balances, credit memo numbers and balances).
    - Resolution outcome: Resolved, Resolved with Write-Off, or Not Resolved (Error).
    - If resolved: the records applied/associated, amounts settled, and any write-off amount.
    - If not resolved: the specific reason(s) why resolution could not be completed (e.g., "No exact program match found", "Multiple programs identified", "Remaining balance of $X exceeds customer tolerance of $Y", "Credit memo currency mismatch").
    - Timestamp of processing.
25. The log must be written back to Oracle as a **note on the claim record**, visible to the AR analyst and Finance manager.
26. The log must be written in plain language that a non-technical user can understand.
27. **Write-off accounting:** When the system performs an automatic write-off for a sub-tolerance balance, it must use **Oracle Channel Revenue Management's write-off mapping** to determine the correct Oracle Receivables activity, configured per business unit. The system must not hard-code write-off reason codes.
28. **Email notification:** The system must send an email notification to the **assigned claim owner (analyst)** for all outcomes — both auto-resolved and flagged-for-review claims. If no claim owner is assigned to the claim, the email must be sent to the **default claim owner configured at the business unit level in Channel Revenue Management**. If no default claim owner is configured at the business unit level, the system must **throw an error and halt processing** for that claim, logging the error as "No claim owner configured for business unit — unable to deliver notification."
29. **In-app notification:** The system must generate an in-app notification for claims that are **flagged for review**. The notification must be visible to **every user who has access to claims**, not only the assigned claim owner.
30. Email and in-app notifications must include the claim identifier, the outcome summary, and a direct link to the claim record.
31. **Trajectory metrics — autonomous resolution:** For every claim resolved autonomously, the system must log a trajectory metric recording whether the resolution was subsequently **approved** (accepted as-is) or **reversed/modified** by an analyst. This metric must be captured as a **structured attribute on the claim record** to support reporting and model improvement.
32. **Trajectory metrics — recommended resolution:** For every claim where the assistant generated a recommendation (rather than autonomous resolution), the system must log a trajectory metric recording whether the analyst **submitted the claim without changes** or **modified the claim beyond notes and attachments** before submission. Changes to notes or attachments alone must not count as a modification for this metric. This metric must also be captured as a **structured attribute on the claim record**.

### 4.6 Customer Configuration

30. The system must derive the **pay-over threshold** from Oracle Channel Revenue Management's native pay-over threshold definition. The threshold is evaluated in the following priority order: **bill-to level → customer level → Channel Revenue Management system setting**. The threshold may be expressed as either a **fixed currency amount** or a **percentage of the claim amount**, as defined in Channel Revenue Management. If no pay-over threshold is configured at any level, the system must default to **zero** (i.e., full accrual coverage is required).
31. Customer **claim tolerance levels** for partial resolution (both promotional and credit memo paths) must be determined by the system using a **customer tolerance RAG document**. The document is structured as a table with entries at either the **customer level or the bill-to level** (bill-to takes precedence when both exist). Each entry specifies a tolerance amount. The document is owned by the **customer** and maintained by the **Claim Analyst Supervisor**. It must be updatable without a code change.
32. If a customer or bill-to has no tolerance entry in the RAG document, the system must default to a tolerance of **zero** (i.e., the claim must be fully covered with no write-off permitted).
33. Both the Channel Revenue Management pay-over configuration and the tolerance RAG document must be resolvable at runtime without redeployment.

### 4.7 Claim Scope

30. The MVP must support **deductions only** as the claim source type. Other claim source types (e.g., overpayments) must be gracefully bypassed, logged as out of scope, and routed to analyst review.

---

## 5. Non-Goals (Out of Scope)

- Resolution of non-deduction claim sources (e.g., overpayments) — planned for a future release.
- Fuzzy or partial matching of program codes or credit memo numbers — exact match only for MVP.
- Automated outreach to customers or vendors (e.g., sending dispute letters).
- Integration with any system other than Oracle Fusion ERP (Receivables) and Oracle Fusion SCM Channel Revenue Management.
- Creation of new credit memos, accruals, or programs — the assistant may only apply or associate existing records.
- Multi-currency deduction resolution (unless already handled natively by Oracle).
- User authentication and access control (assumed to be handled by existing Oracle Fusion security).

---

## 6. Design Considerations

- The assistant should present a **claim-centric workspace**: the analyst sees the deduction details, the assistant's clue-gathering findings, classification result, and resolution outcome (or reason for non-resolution) in a single view.
- Resolution outcomes should use clear, non-technical status labels (e.g., "Resolved via Promotion", "Resolved via Credit Memo", "Needs Your Review — Insufficient Accruals").
- The resolution log should be displayed as a readable activity timeline on the claim, not as a raw data dump.
- Avoid surfacing Oracle internal field names (e.g., flexfield segment codes) directly to the analyst — use business-friendly labels.

---

## 7. Technical Considerations

- The assistant will need **read and write API access** to Oracle Fusion SCM Channel Revenue Management (claims, programs, accruals, write-off mappings, pay-over thresholds) and **read/write access** to Oracle Fusion ERP Receivables (credit memos, receipt applications).
- All Oracle Fusion API calls must respect existing data security and user-level entitlements.
- The **RAG knowledge documents** (claim classification, customer tolerance, and confidence thresholds) must be stored in a retrievable format (e.g., a vector store or indexed document service). The retrieval mechanism must be decoupled from the AI prompts so each document can be updated independently without a code change.
- **RAG confidence thresholds** are resolved at runtime from the confidence threshold RAG document, using the most specific matching entry (customer + settlement method + RAG use case) and falling back to the system-wide default of 90% where no specific entry exists.
- The assistant must be **stateless per claim evaluation** — each deduction is assessed independently with no dependency on prior assistant sessions.
- **Trajectory metrics** must be captured as structured, queryable data to support ongoing model evaluation and improvement. The data model should support future aggregation and reporting.
- Error handling must gracefully manage Oracle API timeouts, missing data, and unexpected claim structures — failures must be logged, not silently swallowed.
- The resolution engine should be designed with extensibility in mind so that future claim sources (overpayments, etc.) can be added as new resolution paths without restructuring the core framework.

---

## 8. Success Metrics

| Metric | Description | Initial Target |
|---|---|---|
| **Auto-settlement rate** | % of claims with at least one program or credit memo clue found that are fully resolved autonomously without analyst intervention | ≥ 70% of claims with found clues |
| **Autonomous resolution approval rate** | % of auto-resolved claims that are subsequently approved without reversal or modification (trajectory metric) | Baseline to be established at launch |
| **Recommendation acceptance rate** | % of recommended resolutions submitted by the analyst without modification beyond notes and attachments (trajectory metric) | Baseline to be established at launch |
| **Classification accuracy** | % of claims correctly classified for type and reason vs. analyst review | ≥ 90% |
| **Average processing time** | Average analyst time spent per deduction (before vs. after) | ≥ 40% reduction |
| **Settlement cycle time** | Average days from deduction creation to resolution | ≥ 30% reduction |
| **Error/reversal rate** | % of auto-resolved deductions later reversed or corrected | < 2% |
| **Log completeness** | % of processed claims with a complete resolution log entry | 100% |
| **Analyst adoption** | % of analysts using the assistant vs. processing manually | ≥ 80% within 90 days of launch |

> Note: Baseline measurements should be captured before go-live to enable meaningful before/after comparison.

---

## 9. Open Questions

All clarifying questions raised during PRD development have been resolved. No open questions remain at this time. New questions may be raised during technical design or sprint planning.
