# PRD Template (Extended)

Extends the `to-prd` house template. Field order is mandatory. The **Provenance** and **Disposition Rationale** sections are mandatory and may not be omitted. The skill writes one populated copy per slice, at `docs/prd/<flow>-<slice-name>.md` in the modernization project root.

The illustrative rows and blocks below show structure and required fields. They are not starting state. Remove all rows marked *(example — remove this row)* and all blocks marked *(example — remove this block)* before committing the file.

---

# PRD: `<flow>` — `<slice-name>`

## Problem Statement

One paragraph from the user's perspective. Describe the business pain or gap the legacy flow encodes, as experienced by the end user — not the technical debt.

*The loan-origination applicant flow currently requires branch staff to re-enter applicant data in three separate screens because the legacy JSP app does not maintain state across form submissions, causing errors and rework on roughly 12% of applications. (example — remove this block)*

---

## Solution

One paragraph from the user's perspective. Describe what the modernized slice delivers — the SPA screens and the Spring Boot services that back them.

*A single-page application wizard guides the applicant through the submission in one continuous session, backed by a `lending-service` endpoint that persists state atomically and wraps the downstream ESB adapters; branch staff submit once and the system handles retries transparently. (example — remove this block)*

---

## User Stories

Long numbered list. Format: `As an <actor>, I want a <feature>, so that <benefit>`. Cover every presentation-surface entry in the target map slice and every edge case extracted from the LBP flow file. Err toward exhaustive.

1. As a loan officer, I want a multi-step application wizard that saves progress automatically, so that I can resume an interrupted session without re-entering data. *(example — remove this row)*
2. As a loan officer, I want inline validation on the income field, so that I discover formatting errors before submitting to the backend. *(example — remove this row)*
3. As a loan officer, I want a clear error message when the credit-check ESB call fails, so that I can tell the applicant what to do next rather than seeing a blank screen. *(example — remove this row)*

---

## Implementation Decisions

Concrete template slots, services, ESB adapters, and shareable domains used in this slice. Reference specific manifest paths. No code snippets; no file paths likely to change.

- SPA module: `src/modules/loan-origination` — new module from the SPA template's multi-step-wizard slot. *(example — remove this row)*
- Spring Boot service: `lending-service` (existing) — extend with `POST /lending/application` endpoint backed by `LoanApplicationHandler`. *(example — remove this row)*
- ESB adapter: `CreditCheckEsbAdapter` wraps `ESB-LOAN-003`; retry policy per the Spring Boot template's adapter slot conventions. *(example — remove this row)*
- Shareable domain: `applicant-identity` at `domains/applicant-identity`; provides `ApplicantDto` used by both the SPA and the service. *(example — remove this row)*
- Manifest citations: SPA manifest § multi-step-wizard; Spring Boot manifest § adapters, § endpoints. *(example — remove this row)*

---

## Testing Decisions

What to test (external behavior only), which modules to cover, and prior art for similar tests.

- Test the `lending-service` endpoint against a stubbed `CreditCheckEsbAdapter`; verify the response shape and HTTP status codes for success, validation failure, and ESB timeout. *(example — remove this row)*
- Test the SPA wizard's state persistence by simulating a mid-session page reload and confirming field values are restored from session storage. *(example — remove this row)*
- Prior art: `AccountSummaryController` integration tests in `lending-service` — same fixture pattern applies. *(example — remove this row)*
- Edge cases from the LBP to cover: partial-submission replay (undocumented behavior flagged in `legacy/flows/loan-origination.md` § Undocumented behaviors), income field with non-ASCII characters. *(example — remove this row)*

---

## Out of Scope

What this PRD does NOT cover. Particularly important: which other slices in the same flow are deferred to other PRDs.

- Credit-decision display (deferred to `loan-origination-decision` PRD). *(example — remove this row)*
- Document upload for supporting evidence (deferred to `loan-origination-documents` PRD). *(example — remove this row)*
- Mainframe-side rule changes — disposition for all rules in this slice is `stay`; Spring Boot provides adapters only. *(example — remove this row)*

---

## Provenance

Mandatory. Points to the authoritative source artifacts for this PRD. Do not omit.

| Artifact | Path |
|---|---|
| Target map | `target/loan-origination-map.md` *(example — remove this row)* |
| LBP flow file | `legacy/flows/loan-origination.md` *(example — remove this row)* |
| Inventory | `legacy/inventory.md` *(example — remove this row)* |
| SPA manifest | `<path declared in target-templates.yaml>` *(example — remove this row)* |
| Spring Boot manifest | `<path declared in target-templates.yaml>` *(example — remove this row)* |

---

## Disposition Rationale

Mandatory. One block per business rule covered by this PRD. Rubric traces are copied verbatim from the target map. This section is the regulator-facing audit trail for every business-rule decision in this slice.

---

### Rule: `<rule-id>` — `<one-sentence rule statement>` *(example — remove this block)*

**Source provenance:** `legacy/flows/<flow>.md` § Business rules § `<rule-id>` *(example — remove this block)*

**Rubric trace:**

| Dimension | Score | Reason |
|---|---|---|
| Deploy cadence | `stay-leaning` | Rule has not changed in three years per comment history. *(example — remove this row)* |
| Regulatory pinning | `stay-leaning` | Cited in the bank's annual regulator submission. *(example — remove this row)* |
| Data gravity | `stay-leaning` | Operates on mainframe-resident account balances. *(example — remove this row)* |
| Performance envelope | `stay-leaning` | Batch-coupled; mainframe throughput required. *(example — remove this row)* |
| Blast radius | `stay-leaning` | Incorrect behavior produces incorrect system-of-record entries. *(example — remove this row)* |
| Legacy-side test coverage | `stay-leaning` | Covered by mainframe regression suite. *(example — remove this row)* |

**Disposition:** `stay`

**Justification:** *(required for non-`stay`; omit for `stay` — the rubric trace is sufficient. For `split`: also state which portion moves and which stays.)*

**Target work:** Spring Boot adapter wraps ESB call `<call-id>`. No Spring Boot implementation of the rule. *(example — remove this block)*

---

*This section starts empty. Add one block per business rule in scope for this slice.*

---

## Further Notes

*Include only if there is genuinely additional context. Omit the section if empty — do not pad with boilerplate.*

Anything else relevant to this PRD: open questions from the target map not yet resolved, assumptions made during slicing, or pointers to related artifacts.

*The target map flags one open question about the credit-score display format (target map § Open questions row 2) — resolve with the SPA template maintainer before implementation begins. (example — remove this block)*
