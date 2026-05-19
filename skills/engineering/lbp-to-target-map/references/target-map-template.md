# Target Map Template

The skill writes one populated copy of this template per flow, at `target/<flow>-map.md` in the modernization project root. Section order is mandatory. Section headings are mandatory. Provenance fields are mandatory in every section — an entry without provenance is inadmissible. The downstream `target-map-to-prd` skill parses by section name; do not rename sections.

The illustrative rows below show structure and required fields. They are not starting state. Remove all rows marked *(example — remove this row)* before committing the file.

---

# Target Map: `<flow>`

## 1. Map header

| Field | Value |
|---|---|
| Flow name | `<flow>` |
| Source flow file | `legacy/flows/<flow>.md` |
| target-templates.yaml | `target-templates.yaml` (path from project root) |
| Manifests consulted | SPA: `<path>`; Spring Boot: `<path>`; Shareable domains: `<name>` at `<path>` |
| Date | YYYY-MM-DD |
| Disposition summary | stay: N / move: N / split: N |

---

## 2. SPA work

One row per presentation-surface entry from the flow file. Source pointer is mandatory — cite the section and entry ID from `legacy/flows/<flow>.md`.

| Source (flow file ref) | Slot (manifest) | Files to create | Modules to extend | API client surface | Manifest citation |
|---|---|---|---|---|---|
| Presentation surface § `<entry-id>` *(example — remove this row)* | `src/modules/loan-origination/pages/ApplicationForm.page.tsx` (form slot) *(example — remove this row)* | `ApplicationForm.page.tsx`, `applicationForm.schema.ts` *(example — remove this row)* | `src/modules/loan-origination/index.ts` *(example — remove this row)* | `GET /api/loan/application/:id`, `POST /api/loan/application` *(example — remove this row)* | SPA manifest § forms *(example — remove this row)* |

*This section starts empty. Add one row per presentation-surface entry in the flow file.*

---

## 3. Spring Boot work

One row per controller or integration surface entry from the flow file.

| Source (flow file ref) | Target service | ESB adapter additions | Endpoints exposed | Security policy hooks | Observability hooks | Manifest citation |
|---|---|---|---|---|---|---|
| Integration surface § `<entry-id>` *(example — remove this row)* | `lending-service` (existing) *(example — remove this row)* | `LoanDecisionEsbAdapter` *(example — remove this row)* | `POST /lending/loan-decision` *(example — remove this row)* | `@RequiresRole("LOAN_OFFICER")` *(example — remove this row)* | `LoanDecisionMeter` (Micrometer) *(example — remove this row)* | Spring Boot manifest § adapters *(example — remove this row)* |

*This section starts empty. Add one row per controller/integration surface entry in the flow file.*

---

## 4. Business rules and dispositions

One block per business rule from the flow file. The rubric trace is mandatory for every rule, including default-`stay` rules. Justification is mandatory for non-`stay` dispositions.

---

### Rule: `<rule-id>` — `<one-sentence rule statement>` *(example — remove this block)*

**Source provenance:** `legacy/flows/<flow>.md` § Business rules § `<rule-id>` — originally observed at `<servlet-class>:<line-range>` *(example — remove this block)*

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

**Justification:** *(required only for non-`stay`; omit for `stay` — the rubric trace is sufficient)*

**Target work:** Spring Boot adapter wraps ESB call `<call-id>`. No Spring Boot implementation of the rule. *(example — remove this block)*

---

*This section starts empty. Add one block per business rule in the flow file.*

---

## 5. Cross-cutting

Anything that touches multiple slices: shared types, shared security policies, shared ESB adapters used by more than one rule or surface entry.

| Item | Kind | Affected rules / entries | Notes |
|---|---|---|---|
| `LoanApplicationDto` *(example — remove this row)* | Shared type *(example — remove this row)* | SPA § form entry, Spring Boot § endpoint *(example — remove this row)* | Defined in `lending-api-contracts`; both sides depend on it. *(example — remove this row)* |

*This section starts empty. Add one row per cross-cutting concern.*

---

## 6. Open questions

Manifest gaps (per `references/manifest-contract.md`), flow-file ambiguities surfaced during mapping, and decisions deferred to the user. Each item must state: what the gap is, which manifest or flow entry it came from, and what the user or manifest author needs to resolve it.

| # | Source | Gap description | Resolution needed from |
|---|---|---|---|
| 1 | SPA manifest *(example — remove this row)* | No slot defined for multi-step wizard flow — `loan-origination` has a 4-step application wizard that does not match any declared manifest slot. *(example — remove this row)* | SPA template maintainer — add a wizard slot, or confirm the nearest equivalent slot and its conventions. *(example — remove this row)* |

*This section starts empty. Add one row per open question.*
