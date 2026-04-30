# PRD Template

Every PRD in `prds/` uses this exact structure. Match the section order. Skip a section only if it genuinely doesn't apply to the slice — and note the skip in `Further Notes`.

---

```markdown
# PRD <N>: <Slice Title>

**Slice ID:** <N>
**Depends on:** <list of PRD IDs, or "none">
**Covers CSP sections:** <comma-separated list of CSP file + section refs>

## Problem Statement

What this slice exists to deliver, framed from the user's or the calling system's perspective. One short paragraph. Not "we need to implement module X" — instead "users need to be able to submit orders and receive a receipt".

## Solution

What this slice delivers, also from the user's or the calling system's perspective. One short paragraph. The build team should be able to read Problem + Solution and immediately understand the slice's value.

## User Stories

A numbered list of user stories, in the format:

> As a `<role>`, I want `<capability>`, so that `<benefit>`.

Cover every externally observable behavior the slice must satisfy. Cite the CSP section that establishes each behavior:

> 1. As an authenticated customer, I want to submit an order, so that I receive a receipt confirming the purchase. *(see `csp/03-behaviors.md` §Order Submission)*
> 2. As an authenticated customer, I want to receive a clear error when funds are insufficient, so that I can correct the issue. *(see `csp/05-effects-and-errors.md` §Payment Errors)*

If the slice has many stories (>10), group them under sub-headings.

## Implementation Decisions

A list of decisions the slice depends on. Pull these from the CSP — do not invent. Include:

- Modules to build or modify in this slice (cite `csp/01-architecture.md`)
- The public contract each module must honor (cite `csp/02-public-api.md`)
- Data shapes the slice needs (cite `csp/06-data-model.md`)
- Effects the slice produces or consumes (cite `csp/05-effects-and-errors.md`)
- Architectural boundaries the slice must respect

Do NOT include source code, source-language type syntax, or source file paths. Use the CSP's pseudo-syntax notation. Do NOT prescribe an implementation algorithm; describe the contract and let the build team design.

If the parent CSP was generated with a target paradigm, you may reference it here at the contract level (e.g. "the module exposes effectful operations returning typed errors") — but never at the framework level (no library names, no specific runtime primitives).

## Testing Decisions

What the build team must test, framed as required guarantees. Pull from `csp/07-test-contracts.md`:

- Which black-box scenarios from `csp/07-test-contracts.md` apply to this slice (cite by scenario name or section)
- What additional test coverage the slice needs that the CSP doesn't already specify (e.g. integration tests across modules introduced together in this slice)
- A reminder: tests verify externally observable behavior, not implementation choices

If the slice introduces a module boundary that isn't otherwise testable end-to-end without dependent slices, note it.

## Out of Scope

A short list of behaviors, modules, or concerns explicitly NOT covered by this slice — typically because they're picked up by a later PRD. Be specific:

> - User authentication: handled by PRD 01-foundations
> - Refund flows: handled by PRD 04-refunds
> - Payment provider integration beyond the abstract contract: handled by PRD 03-payments-adapter

This section is what makes the slice "independently grabbable" — without it, scope creeps.

## Open Questions

CSP `[INTENT UNCLEAR]` flags or other ambiguities the build team must resolve. Frame each as a decision they need to make:

> - The CSP marks the retry policy for failed webhooks as `[INTENT UNCLEAR]`. Build team to choose: bounded exponential backoff, fixed retry, or no retry. Recommend documenting the chosen policy in an ADR.

If there are no open questions, write "None." Don't omit the section.

## Further Notes

Anything else the build team should know that doesn't fit elsewhere. Cross-references to other PRDs, links to relevant external standards (e.g. "the wire format is JSON-LD per W3C spec"), or notes on why a section was skipped.
```

---

## Filling the template — what good looks like

**Problem Statement and Solution**: read like a brief stakeholder pitch, not a technical brief. If they sound like specs, they're in the wrong section.

**User Stories**: cite the CSP. Every story should be traceable. A story without a citation is either an invention (delete) or evidence of a CSP gap (escalate to the user).

**Implementation Decisions**: prescribe contracts, not code. "The Payment module must expose `submit(order, idempotencyKey) -> Result<Receipt, SubmitError>`" is a contract. "Use a `case class Receipt(...)` with…" is contamination. Stay neutral.

**Testing Decisions**: name the scenarios; don't transcribe them. The build team reads `csp/07-test-contracts.md` for the actual scenarios.

**Out of Scope**: be aggressive. Anything that isn't unambiguously in scope is out. The other PRDs will catch what this one drops.

**Open Questions**: never silently resolve a CSP `[INTENT UNCLEAR]` flag. Surface it. The build team makes the call, ideally documenting it in an ADR.
