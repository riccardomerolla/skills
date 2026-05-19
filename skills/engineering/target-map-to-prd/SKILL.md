---
name: target-map-to-prd
description: Slice a target map produced by lbp-to-target-map into vertical-slice PRDs in the bank's house PRD template, extended with provenance and strangler-fig disposition rationale, ready to feed into the existing to-issues skill. Use when the user wants to convert a target map to PRDs, slice modernization work into independently-grabbable tickets, or says "PRDs from this map", "ticket slices for loan-origination", or "produce the build-team handoff for this flow".
---

# Target Map to PRD

Slices a target map into vertical-slice PRDs. Each PRD follows the bank's house template (the same fields `to-prd` uses) extended with two mandatory sections — Provenance and Disposition Rationale — that carry the regulator-readable audit trail forward from the target map. Writes files to disk and stops. The user reviews them and invokes `to-issues` to create tickets.

## Preconditions

- `target/<flow>-map.md` exists. If it does not, stop and tell the user to run `lbp-to-target-map` first.
- The bank's house PRD template as encoded in `references/prd-template.md` is the format target.

## Process

1. Read `target/<flow>-map.md`. Identify natural vertical slices: typically one slice per flow, but a wide flow with multiple bounded contexts produces multiple slices — each spanning SPA work, Spring Boot work, and associated business rules end-to-end.

2. For each slice, draft a PRD using `references/prd-template.md`:
   - **Problem Statement** — from the LBP flow's purpose and user-facing pain.
   - **Solution** — from the target map's SPA and Spring Boot work for this slice.
   - **User Stories** — one per presentation-surface entry; expand each with edge cases extracted from the LBP flow file.
   - **Implementation Decisions** — concrete template slots, services, ESB adapters, shareable domains; cite manifest paths.
   - **Testing Decisions** — informed by the LBP's edge cases and undocumented behaviors; cover external behavior only.
   - **Out of Scope** — which other slices in the same flow are deferred to other PRDs.
   - **Provenance** — pointers to the target map, the source LBP flow file, and the inventory.
   - **Disposition Rationale** — every business rule in scope: rule statement, full rubric trace, disposition, justification if non-`stay`.

3. Write each PRD to `docs/prd/<flow>-<slice-name>.md` at the modernization project root.

4. Do not publish. Tell the user the PRDs are written and where they live. Suggest reviewing them, then invoking `to-issues` for ticket creation.

## Distinction from to-prd

- `to-prd` takes conversation context; this skill takes a target map file.
- `to-prd` auto-publishes to the issue tracker; this skill writes to disk and stops.
- `to-prd` does not include Provenance or Disposition Rationale; this skill does — they are non-negotiable.

## Provenance discipline

Every PRD includes a Provenance section pointing back to the target map file, the source LBP flow file, and the inventory. Every Disposition Rationale section copies the rubric traces from the target map verbatim. These two sections are non-negotiable; they are the regulator-readable audit trail for every business-rule decision in the slice.

## What this skill is not

- Not a publisher — use `to-issues` to create tickets from the PRDs.
- Not an interviewer — use `to-prd` for conversation-context-driven PRDs.
- Not a code generator.

## See also

- `references/prd-template.md` — the extended PRD template this skill populates
- `../to-issues/SKILL.md` — ticket creation from PRDs (next step)
- `../to-prd/SKILL.md` — conversation-driven PRD with auto-publish
- `../legacy-inventory/SKILL.md` — whole-app inventory (first step in the pipeline)
- `../legacy-extract-flow/SKILL.md` — per-flow deep extract
- `../lbp-to-target-map/SKILL.md` — produces the target map this skill reads
