# Disposition Rubric

This rubric is applied to every business rule in a flow. The default disposition is `stay` — the rule remains on the mainframe and Spring Boot provides an adapter. `move` (rule migrates to Spring Boot) and `split` (rule partially moves, partially stays) are exceptions. The rubric trace is recorded for every rule including default-`stay` rules — this uniform audit trail is the regulator artifact for every boundary decision.

## The six dimensions

Each dimension is scored as `stay-leaning`, `neutral`, or `move-leaning` with a one-line reason that cites the flow file.

### 1. Deploy cadence

*How often does this rule change?*

- `stay` signal: rare changes, or changes aligned with the mainframe quarterly release.
- `move` signal: monthly or more frequent business-driven changes that the mainframe release calendar cannot accommodate.

### 2. Regulatory pinning

*Is the rule pinned by regulation or audit?*

- `stay` signal: rule cited in regulator reviews, present in audit-controlled COBOL, central to the mainframe system of record.
- `move` signal: rule is purely UX or workflow logic with no regulatory pinning.

### 3. Data gravity

*Where does the data the rule operates on live?*

- `stay` signal: rule depends on mainframe-resident data that would require expensive replication or synchronization.
- `move` signal: rule operates on data already available outside the mainframe, or on data that is naturally Spring-Boot-side (session, request, derived).

### 4. Performance envelope

*Does the rule need mainframe-class performance?*

- `stay` signal: high-volume, low-latency, batch-coupled, or transactionally coupled to mainframe operations.
- `move` signal: workflow-paced, user-facing, or low-volume.

### 5. Blast radius

*What happens when this rule breaks?*

- `stay` signal: incorrect behavior affects regulated outputs, system-of-record entries, or cross-system integrity.
- `move` signal: incorrect behavior is recoverable, user-visible-only, or contained to one service.

### 6. Legacy-side test coverage

*How well-tested is the existing rule on the mainframe?*

- `stay` signal: rule is heavily tested on the mainframe side; moving forfeits that coverage.
- `move` signal: rule has weak or no mainframe tests, OR the modernization will add stronger tests Spring-Boot-side.

## Decision procedure

1. Score each dimension as `stay-leaning`, `neutral`, or `move-leaning` with a one-line reason citing the flow file.
2. If all six are `stay-leaning` or `neutral`: disposition is `stay`. No justification required beyond the trace.
3. If three or more are `move-leaning`: disposition is `move`. Justification required — name the dimensions that drove the exception.
4. If one or two are `move-leaning` and the rest are mixed: this is a `split` candidate. Justification required — state explicitly which portion moves and which stays, and why splitting is preferable to a clean `stay` adapter.
5. The rubric trace and disposition are written into the target map's business-rules section. Always.

## Examples

### Example A — `stay` (interest accrual rule)

| Dimension | Score | Reason |
|---|---|---|
| Deploy cadence | stay-leaning | Changes only with regulatory rate reviews, quarterly at most. |
| Regulatory pinning | stay-leaning | Cited in audit reports; COBOL paragraph is in audit-controlled source. |
| Data gravity | stay-leaning | Accrual data lives in mainframe VSAM; replication cost is prohibitive. |
| Performance envelope | stay-leaning | Batch-coupled overnight run; mainframe throughput required. |
| Blast radius | stay-leaning | Incorrect accrual produces incorrect system-of-record balances. |
| Legacy-side test coverage | stay-leaning | Comprehensive regression suite on the mainframe. |

Disposition: `stay`. Spring Boot adapter wraps the ESB call; no Spring Boot implementation of the rule.

### Example B — `move` (promotional eligibility rule)

| Dimension | Score | Reason |
|---|---|---|
| Deploy cadence | move-leaning | Marketing campaigns change eligibility weekly; mainframe release cadence blocks updates. |
| Regulatory pinning | move-leaning | UX decision with no regulatory citation found in flow file. |
| Data gravity | move-leaning | Eligibility criteria operate on session data and product catalogue already in Spring Boot. |
| Performance envelope | move-leaning | User-facing, single request, no batch coupling. |
| Blast radius | move-leaning | Incorrect eligibility is recoverable; customer support can override. |
| Legacy-side test coverage | move-leaning | No mainframe tests found for this rule; flow file notes behavior is inferred from servlet logic. |

Disposition: `move`. Justification: all six dimensions are move-leaning — deploy cadence is the primary driver (marketing cannot wait for quarterly mainframe releases). Spring Boot implements the rule; ESB call is retired once the migration is validated.

### Example C — `split` (credit limit validation)

| Dimension | Score | Reason |
|---|---|---|
| Deploy cadence | stay-leaning | Core limit calculation is stable; UI display format changes frequently. |
| Regulatory pinning | stay-leaning | Limit calculation is audit-controlled. |
| Data gravity | stay-leaning | Customer credit data is mainframe-resident. |
| Performance envelope | neutral | Single request; mainframe performance not required, but available. |
| Blast radius | stay-leaning | Limit errors affect system-of-record; display errors are UX-only. |
| Legacy-side test coverage | move-leaning | Display-formatting logic has no mainframe test; only the calculation is covered. |

Disposition: `split`. Justification: one dimension is move-leaning (display format logic has no coverage and changes frequently). The calculation and system-of-record write stay on the mainframe; Spring Boot adapts that call. The display-formatting and threshold-messaging logic moves to Spring Boot where it can be tested and iterated independently.
