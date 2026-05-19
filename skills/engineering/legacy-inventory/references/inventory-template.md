# Inventory Output Template

This is the template for the inventory output. The skill writes a populated copy to `legacy/inventory.md` at the modernization project root. Section order is mandatory; section headings are mandatory; table structure within catalogue sections is mandatory (downstream skills parse columns by name); bullet structure within non-table sections is recommended.

The illustrative rows below show structure and required fields. They are not starting state. Remove all rows marked *(example — remove this row)* and all blocks marked *(example — remove this block)* before committing the file.

---

## 1. Overview

| Field | Value |
|---|---|
| System name | |
| Source repo path | |
| Date of scan | |
| Scan scope | repo root, modules included, excluded paths |

One-paragraph summary of the system: what it does, the business domain it serves, its primary users, and the rough scale (number of JSPs, servlets, ESB destinations).

---

## 2. JSP Catalogue

| JSP path | One-line purpose | Form posts to (servlet) | Provenance (file:line) |
|---|---|---|---|
| `web/pages/AccountSummary.jsp` | Displays account balance and recent transactions | `AccountSummaryServlet` | `web/pages/AccountSummary.jsp:23-45` *(example — remove this row)* |

Include every JSP found under `webapp/` or equivalent. Infer purpose from page title, `<h1>`, form labels, and field names — do not transcribe HTML.

---

## 3. Servlet Catalogue

| Servlet class | URL pattern(s) | JSPs dispatched to | ESB calls invoked | Provenance |
|---|---|---|---|---|
| `com.bank.web.AccountSummaryServlet` | `/account/summary` | `AccountSummary.jsp` | `ESB-ACCT-001` | `src/.../AccountSummaryServlet.java:1` *(example — remove this row)* |

Derive URL patterns from `web.xml` mappings. Derive ESB calls from the servlet body or the service class it delegates to.

---

## 4. ESB Call Surface

| Call ID | Destination kind | Destination name | Invoking servlets | Rough payload shape | Provenance |
|---|---|---|---|---|---|
| `ESB-ACCT-001` | Transaction code | `ACCTBAL` | `AccountSummaryServlet` | account number in, balance + transactions out | `src/.../AccountService.java:42` *(example — remove this row)* |

Destination kind is one of: `transaction code`, `MQ queue`, `CICS program`, `other`. Assign a stable call ID (e.g. `ESB-<DOMAIN>-<NNN>`) so downstream skills can reference it.

---

## 5. Mainframe Contract Surface

| Contract ID | Transaction code or program | Direction | Invoked from (ESB call IDs) | Known semantics | Provenance |
|---|---|---|---|---|---|
| `MC-ACCT-001` | `ACCTBAL` | out | `ESB-ACCT-001` | Returns current balance and last 10 transactions for an account | `mule-config.xml:88` *(example — remove this row)* |

Direction: `in` (mainframe initiates), `out` (J2EE initiates), `both`. If semantics are not visible from the ESB contract, write `opaque`. Do not open COBOL to fill this field — that is `legacy-extract-flow`'s job on targeted probes.

---

## 6. Domain Glossary

| Term | Observed in | Candidate canonical name | Notes |
|---|---|---|---|
| `AcctBal` | JSP labels, ESB payload fields | `AccountBalance` | Abbreviated consistently; expand in domain model *(example — remove this row)* |

Sources for "observed in": JSP labels, form field names, servlet names, ESB payload keys, `web.xml` display names.

---

## 7. Candidate Bounded Contexts

For each candidate, use this structure:

**Context name:** `<CamelCase name>` *(example — remove this block)*

- **Included JSPs:** list of JSP paths
- **Included servlets:** list of servlet classes
- **Included ESB calls:** list of call IDs
- **Included contract IDs:** list of contract IDs
- **Rationale:** One paragraph explaining what clusters these components together — shared form-field vocabulary, co-occurring ESB calls, shared data objects, common user journey.

If no clear clustering emerges, propose zero contexts and record the ambiguity in section 8 — do not invent contexts to fill the section. If more than six candidates surface, group the smallest related ones under a single "Unclassified" candidate rather than proliferating entries — downstream skills consume this list as a starting point for flow extraction, not an exhaustive map.

Mark every context **candidate**. Boundaries are not final until `lbp-to-target-map` confirms them against the target architecture templates.

---

## 8. Open Questions

List anything ambiguous the scan turned up. Typical entries:

- Orphan JSPs (no servlet references them).
- Servlets with no JSP callers (possibly called programmatically or via redirect from another servlet).
- ESB calls with no invoking servlet visible in source (possibly invoked from a scheduled job or batch process outside this repo).
- Domain glossary terms with conflicting meanings across JSPs or ESB payloads.
- Modules excluded from the scan that may contain additional servlets or JSPs.

---

## Provenance reminder

Every catalogue entry must carry a provenance citation (file path and, where applicable, line range). The inventory is a regulator artifact. Entries without provenance are inadmissible.
