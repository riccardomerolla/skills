# Flow Output Template

The `legacy-extract-flow` skill writes one populated copy of this template to `legacy/flows/<flow>.md` at the modernization project root. Section order is mandatory; section headings are mandatory; table structure within catalogue-style sections is mandatory — downstream skills (`lbp-to-target-map`) parse columns by name. Provenance fields are mandatory in every catalogue row; a row without provenance is inadmissible. The flow file is a regulator artifact.

---

## 1. Flow Header

| Field | Value |
|---|---|
| Flow name | |
| Extraction date | |
| Source repo path | |
| Inventory file | `legacy/inventory.md` |
| Scan scope | List of JSPs, servlets, and ESB call IDs considered in-scope for this flow |

---

## 2. Presentation Surface

One row per JSP that participates in this flow.

| JSP path | Purpose | Form fields (name · type · client-side validation) | Navigation links | Posts to (servlet) | Provenance (file:line) |
|---|---|---|---|---|---|
| `web/pages/LoanEntry.jsp` | Collects loan application data | `loanAmount · number · required,min=1`; `term · select · required` | `/loan/review` | `LoanEntryServlet` | `web/pages/LoanEntry.jsp:18-94` |

List every form field as `<name> · <type> · <validation rules>`. Infer purpose from page title, `<h1>`, and field labels — do not transcribe HTML.

---

## 3. Controller Surface

One row per servlet that participates in this flow.

| Servlet class | URL pattern | Request shape | Response shape | JSPs dispatched | ESB calls invoked | Provenance |
|---|---|---|---|---|---|---|
| `com.bank.loan.LoanEntryServlet` | `/loan/entry` | `loanAmount: long`, `term: int`, `customerId: string` | Redirect to `/loan/review` on success; redisplay form on error | `LoanEntry.jsp`, `LoanError.jsp` | `ESB-LOAN-001` | `src/.../LoanEntryServlet.java:1` |

Request and response shapes are inferred from `request.getParameter()` calls and `RequestDispatcher` / redirect targets. Do not transcribe source code.

---

## 4. Integration Surface

One row per ESB call in-scope for this flow. Call IDs must match the inventory.

| Call ID | Inferred request payload | Inferred response payload | Trigger conditions | Destination contract ID | Provenance |
|---|---|---|---|---|---|
| `ESB-LOAN-001` | `customerId: string`, `loanAmount: long`, `termMonths: int` | `approvalStatus: enum(APPROVED,DECLINED,PENDING)`, `referenceId: string` | Called after form validation passes, before redirect | `MC-LOAN-001` | `src/.../LoanService.java:67-81` |

Payload shapes are inferred from the servlet and ESB configuration — not from COBOL source unless a probe was performed (record probed fields in section 10).

---

## 5. Business Rules

One row per rule encoded in this flow.

| Rule ID | Rule statement (plain language) | Provenance | Trigger conditions | Observed examples | COBOL-probed |
|---|---|---|---|---|---|
| `BR-LOAN-001` | Loan amount must be between 1,000 and 500,000 inclusive | `LoanEntry.jsp:34`, `LoanEntryServlet.java:55` | On form submission, before ESB call | Amount=0 → validation error displayed | No |
| `BR-LOAN-002` | Applicants with a delinquency flag receive DECLINED regardless of amount | `LoanEntryServlet.java:82` (trigger), `LNAPPRVL` paragraph `CHKDELQ` lines 320-340 | On ESB-LOAN-001 response processing | — | Yes — see section 10 |

Rule statements are one sentence in plain language. For COBOL-probed rules, provenance includes both the ESB-side trigger and the COBOL location.

---

## 6. Data Flow

Narrative description of data movement through the flow. Include one ASCII sketch or numbered sequence.

**Example sequence:**

```
1. Form fields (loanAmount, term, customerId)
      → [LoanEntry.jsp POST]
2. Servlet validates fields; assembles ESB request
      → ESB-LOAN-001 request payload (customerId, loanAmount, termMonths)
3. ESB routes to mainframe contract MC-LOAN-001 (LNAPPRVL transaction)
      → Mainframe returns (approvalStatus, referenceId)
4. Servlet processes response; sets session attributes
      → Redirect to LoanReview.jsp
5. LoanReview.jsp renders approval status and reference ID to user
```

Replace with the actual flow. Keep it readable — the goal is to trace each piece of data from origin to render.

---

## 7. Edge Cases

| Scenario | Trigger | Observed handling | Provenance |
|---|---|---|---|
| Validation failure on loan amount | `loanAmount < 1000 or > 500000` | Redisplay `LoanEntry.jsp` with inline error message | `LoanEntryServlet.java:55-62` |
| ESB timeout | ESB call exceeds configured timeout | Redirect to `LoanError.jsp` with generic error; no retry | `LoanEntryServlet.java:101-108` |
| Partial ESB failure | `approvalStatus=null` in response | Treated as PENDING; referenceId logged | `LoanEntryServlet.java:93` |

---

## 8. Undocumented Behaviors

Explicit list of things the source does that are not obvious from labels, comments, or contracts. Each item is a candidate for grilling via `grill-with-docs`.

- **Example:** `LoanEntryServlet` sets a session attribute `_loanLock` before the ESB call and clears it on success, suggesting intent to prevent duplicate submissions — but no error path clears the lock on ESB failure. Behavior on re-submission after timeout is unspecified.

List every item observed. Do not omit them because they seem minor — undocumented behaviors are where bugs and regulatory gaps live.

---

## 9. Open Questions

List anything ambiguous the extract turned up.

- Example: `LoanReview.jsp` references a `reviewerId` field not visible in any servlet request or session attribute. Source of this field is unknown.
- Example: ESB call `ESB-LOAN-001` has a `priority` field in the config but no servlet sets it. Default behavior unknown.

---

## 10. COBOL Probe Audit Log

Empty by default. Populated by the COBOL probe procedure (`references/cobol-probe-procedure.md`) when a business rule cannot be specified from J2EE source and ESB contract alone. Every probe is recorded here — this is the regulator-facing audit trail of mainframe access.

| Probe # | ESB call ID | COBOL program | COBOL paragraph | Line range | Rule extracted (Rule ID) | Justification |
|---|---|---|---|---|---|---|
| 1 | `ESB-LOAN-001` | `LNAPPRVL` | `CHKDELQ` | 320–340 | `BR-LOAN-002` | ESB response field `approvalStatus` is set by mainframe; ESB contract lists `DECLINED` as a possible value but does not document the delinquency condition. Rule cannot be specified without reading the mainframe logic. |
