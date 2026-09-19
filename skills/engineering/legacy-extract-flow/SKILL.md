---
name: legacy-extract-flow
description: Deep extract of a single named flow from a legacy J2EE app: JSP forms, servlet routing, ESB calls with payload shapes, business rules with provenance, edge cases, undocumented behaviors. Permitted to perform targeted COBOL probes when a business rule lives mainframe-side and the ESB contract does not specify it. Reads the inventory produced by legacy-inventory. Use when extracting a specific flow ("extract loan-origination", "deep dive on this JSP flow") for modernization, after running legacy-inventory.
---

# Legacy Extract Flow

Deep per-flow extraction from a J2EE codebase. Reads `legacy/inventory.md` as its starting point and accretes output into `legacy/flows/<flow>.md` at the modernization project root. This file is the behavioral specification for one flow: the source of truth downstream skills read when mapping to a target architecture. Never rewrites an existing flow file unless the user explicitly asks.

## Preconditions

- `legacy/inventory.md` must exist. If it does not, stop and tell the user to run `legacy-inventory` first.
- A flow name must be supplied, or chosen from the inventory's candidate bounded contexts. If neither is present, list the candidates and ask the user to pick one.

## Process

1. **Read `legacy/inventory.md`.** Identify the subset relevant to the named flow: JSPs, servlets, ESB call IDs, and mainframe contract IDs that the inventory clusters under this flow's candidate bounded context.

2. **Read those JSP and servlet sources in full.** Extract:
   - Presentation surface: form fields (name, type, client-side validations), navigation links, the servlet each form posts to.
   - Controller surface: URL pattern, request shape, response shape, JSPs dispatched to, ESB calls invoked with trigger conditions.

3. **For each ESB call in the flow,** extract the inferred payload shape (request fields and response fields) and the trigger conditions in the servlet that fire the call.

4. **Enumerate the business rules the flow encodes.** For each rule, attempt to specify it from JSP + servlet + ESB contract alone. If a rule clearly lives mainframe-side and the ESB contract is insufficient to specify it for a Spring Boot adapter to wrap correctly, invoke the **B-mode COBOL probe** (a bounded, read-only mainframe paragraph read; see `references/cobol-probe-procedure.md`). Lift the rule and record the COBOL location as provenance.

5. **Capture data flow end-to-end:** form fields → servlet variables → ESB request payload → mainframe contract → response payload → servlet response → JSP render.

6. **Capture edge cases:** error paths, validation failures, partial-failure handling, idempotency observed.

7. **Flag undocumented behaviors** explicitly. Each item is a candidate for grilling.

8. **Write the flow file** using `references/flow-template.md`. If `legacy/flows/<flow>.md` already exists, stop and ask the user whether to overwrite. Do not proceed with writing until the user explicitly confirms. If confirmed, overwrite the file and record the overwrite date in the Flow Header.

## The COBOL probe boundary

COBOL probes are surgical, recorded, and exceptional. The default path reads only J2EE source. A probe is only justified when a business rule clearly lives mainframe-side and the ESB contract for the relevant call is not expressive enough to specify the rule. Probing is never the default path; defaulting to it destroys the strangler-fig posture and pollutes the regulator artifact. The full procedure is in `references/cobol-probe-procedure.md`.

## Provenance discipline

Every behavior, rule, and contract must cite its source file path and line range. Non-negotiable. For COBOL-probed rules, both the ESB-side trigger and the COBOL paragraph (program + paragraph + line range) are recorded. The flow file is a regulator artifact. Entries without provenance are inadmissible.

## Accretion rule

Subsequent runs add new flow files under `legacy/flows/`. They do not modify existing flow files. To refresh an existing flow file, the user must explicitly ask.

## Grilling

After writing a flow file, suggest the user runs `grill-with-docs` pointed at the new flow file to surface missing rules and undocumented edge cases. Do not invoke `grill-with-docs` automatically.

## What this skill is not

- Not an inventory: that is `legacy-inventory`.
- Not a target architecture mapping: that is `lbp-to-target-map`.
- Not a wholesale COBOL reader: mainframe reads are bounded probes with explicit justification.
- Not a security review.

## See also

- `references/flow-template.md`: output schema and section order for `legacy/flows/<flow>.md`
- `references/cobol-probe-procedure.md`: the B-mode COBOL probe procedure
- `legacy-inventory`: whole-app inventory (run this first)
- `lbp-to-target-map`: maps the Legacy Behavior Pack to the target architecture
- `target-map-to-prd`: turns the target map into implementation PRDs
