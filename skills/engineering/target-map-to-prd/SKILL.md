---
name: target-map-to-prd
description: Slice a target map produced by lbp-to-target-map into vertical-slice PRDs in the bank's house PRD template, extended with provenance and strangler-fig disposition rationale, ready to feed into the existing to-tickets skill. Use when the user wants to convert a target map to PRDs, slice modernization work into independently-grabbable tickets, or says "PRDs from this map", "ticket slices for loan-origination", or "produce the build-team handoff for this flow".
---

# Target Map to PRD

Slices a target map into vertical-slice PRDs. Each PRD follows the bank's house template (the same fields `to-spec` uses) extended with two mandatory sections, Provenance and Disposition Rationale, that carry the regulator-readable audit trail forward from the target map. Writes files to disk and stops. The user reviews them and invokes `to-tickets` to create tickets.

## Preconditions

- `target/<flow>-map.md` exists. If it does not, stop and tell the user to run `lbp-to-target-map` first.
- The bank's house PRD template as encoded in `references/prd-template.md` is the format target.

## Process

1. Read `target/<flow>-map.md`. Then read the flow file at the path named in the target map's Map header (`Source flow file` row) and `legacy/inventory.md`; these carry the user-facing context, edge cases, and undocumented behaviors the PRD must surface. Identify natural vertical slices. Default to one slice per flow. Produce multiple slices only when the target map's SPA work covers more than one distinct user journey, OR its Spring Boot work spans more than one bounded-context service. Each slice must be end-to-end (SPA + Spring Boot + business rules); never split horizontally (e.g. "frontend slice" vs "backend slice").

2. For each slice, draft a PRD using `references/prd-template.md`:
   - **Problem Statement**: from the LBP flow's purpose and user-facing pain.
   - **Solution**: from the target map's SPA and Spring Boot work for this slice.
   - **User Stories**: one per presentation-surface entry; expand each with edge cases extracted from the LBP flow file.
   - **Implementation Decisions**: concrete template slots, services, ESB adapters, shareable domains; cite manifest paths.
   - **Testing Decisions**: informed by the LBP's edge cases and undocumented behaviors; cover external behavior only.
   - **Out of Scope**: which other slices in the same flow are deferred to other PRDs.
   - **Provenance**: pointers to the target map, the source LBP flow file, the inventory, and the relevant template manifests (SPA, Spring Boot, shareable domains).
   - **Disposition Rationale**: every business rule in scope: rule statement, full rubric trace, disposition, justification if non-`stay`.

3. Write each PRD to `docs/prd/<flow>-<slice-name>.md` at the modernization project root.

4. Do **not** publish. Tell the user the PRDs are written and where they live. Suggest reviewing them, then invoking `to-tickets` for ticket creation.

## Distinction from `to-spec`

- `to-spec` takes conversation context; this skill takes a target map file.
- `to-spec` publishes to the issue tracker as its final step; this skill does not publish at all; it writes files and stops.
- `to-spec` does not include Provenance or Disposition Rationale; this skill does, and they are non-negotiable.

## Provenance discipline

Every PRD includes a Provenance section pointing back to the target map file, the source LBP flow file, and the inventory. Every Disposition Rationale section copies the rubric traces from the target map verbatim. These two sections are non-negotiable; they are the regulator-readable audit trail for every business-rule decision in the slice.

## What this skill is not

- Not a publisher: use `to-tickets` to create tickets from the PRDs.
- Not an interviewer: use `to-spec` for conversation-context-driven PRDs.
- Not a code generator.

## See also

- `references/prd-template.md`: the extended PRD template this skill populates
- `to-tickets`: ticket creation from PRDs (next step)
- `to-spec`: conversation-driven PRD with auto-publish
- `legacy-inventory`: whole-app inventory (first step in the pipeline)
- `legacy-extract-flow`: per-flow deep extract
- `lbp-to-target-map`: produces the target map this skill reads
