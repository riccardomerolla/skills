# Manifest Contract

This file specifies the contract between `lbp-to-target-map` and the template manifests it reads. The contract is **not** a schema specification; that lives with llm4zio. This file specifies what the skill *asks* of any manifest. A manifest that satisfies the three questions, in any format, is sufficient.

## The three questions every manifest must answer

### 1. Slot resolution

*Given a behavior of shape X, which slot in this template fills it?*

- **SPA manifest shapes:** "user-facing form", "list view", "detail view", "wizard step", "modal", "API client call". The slot must resolve to a concrete module path and the file conventions for adding new content: file naming, route registration, state management hooks.

- **Spring Boot service-template manifest shapes:** "REST endpoint", "ESB adapter", "domain service", "security policy", "observability hook", "scheduled job". The slot must resolve to a package path and file conventions.

- **Shareable-domain manifest shapes:** "existing capability we can reuse". The slot must resolve to the public API of the shared domain and how to import or depend on it.

### 2. Convention surfacing

*What naming convention, file structure, and base-class/interface conventions apply in slot Y?*

The skill writes target maps that reference concrete file paths and types. Conventions must be machine-readable enough that the skill writes correct paths and names rather than placeholders. Generic guidance ("follow team conventions") is not sufficient; the skill must be able to produce `src/modules/loan-origination/LoanOrigination.page.tsx` or `com.bank.lending.origination.adapter.LoanEsbAdapter`, not `<some file in the right place>`.

### 3. Capability inventory

*What shareable domains are available, and what behaviors do they already cover?*

For every shareable-domain manifest listed in `target-templates.yaml`, the skill must be able to enumerate covered capabilities. A behavior in the flow that matches an existing capability is mapped to that capability, not to a new implementation. The manifest must state this clearly enough that the skill can make the match without guessing.

## Behavior when the contract is unmet

If a manifest cannot answer a question for a specific behavior:

1. The skill records the gap in the target map's **Open questions** section: which manifest, which question, which behavior triggered the gap.
2. The skill emits a recommendation for the manifest authors: what the manifest needs to add to resolve the gap.
3. The skill does not invent missing slots, does not guess naming conventions, and does not assume a capability exists. It stops for that behavior and moves on.

A gap does not fail the entire mapping run; it surfaces as an open question the user must resolve before the target map can be used downstream.

## Note for llm4zio integration

If llm4zio already has a template manifest schema, the skill should align field names with it. If not, the skill treats the three questions above as the canonical contract and surfaces any ambiguity to the user. The contract is intentionally format-agnostic: YAML, JSON, and structured Markdown are all acceptable as long as the three questions can be answered.
