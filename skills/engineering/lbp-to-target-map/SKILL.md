---
name: lbp-to-target-map
description: Map an extracted legacy flow to template-shaped modernization work in the bank's production-ready Next.js SPA template and Spring Boot service template, with strangler-fig disposition by default for every business rule. Reads a flow file from legacy-extract-flow plus template manifests declared at the modernization project root. Use when the user wants to design the target work for a specific extracted flow, or says "map this flow to the templates", "apply strangler-fig to this flow", or "what should we build for loan-origination".
---

# LBP to Target Map

Maps a single extracted flow to template-shaped modernization work in the bank's production-ready Next.js SPA and Spring Boot service templates. Strangler-fig is the default disposition for every ESB-reached business rule — `move` and `split` are exceptions requiring justification. Reads template manifests declared at the modernization project root. Output is `target/<flow>-map.md`.

## Preconditions

- `legacy/flows/<flow>.md` exists. If not, stop and tell the user to run `legacy-extract-flow` first.
- `target-templates.yaml` (or equivalent) at the modernization project root declares:
  - SPA template manifest path
  - Spring Boot service template manifest path
  - Each shareable-domain manifest path

## Manifest contract

The skill expects each referenced manifest to answer three questions: (a) given a behavior of shape X, which slot in this template fills it; (b) what naming convention applies in slot Y; (c) what shareable domains are available and what behaviors they already cover. Full contract in `references/manifest-contract.md`. If a referenced manifest cannot answer all three for a given behavior, the skill flags the gap and stops for that behavior rather than guessing.

## Process

1. Read the flow file. Read the inventory file for cross-references. Read `target-templates.yaml` and the manifest files it points at.
2. For each entry in the flow's presentation surface, map to SPA work using the SPA template manifest: concrete module/page/view paths to create, existing modules to extend, API client surface needed.
3. For each entry in the flow's controller and integration surfaces, map to Spring Boot work using the service-template and shareable-domain manifests: target service (existing, new from template, or shareable domain), ESB adapter additions, endpoints to expose, security policy hooks, observability hooks.
4. For **every** business rule in the flow, run the disposition rubric in `references/disposition-rubric.md`. Default disposition is `stay`. Record the rubric trace (one line per rubric dimension with a brief reason) regardless of disposition. Non-`stay` dispositions REQUIRE an explicit justification field.
5. Write `target/<flow>-map.md` using `references/target-map-template.md`.

## Strangler-fig default

Every ESB-reached business rule is `stay` by default — the rule remains on the mainframe and Spring Boot provides an adapter. `move` and `split` are exceptions that require justification. The rubric is run for every rule including default-`stay` ones so the audit trail is uniform. A non-default disposition is a deliberate engineering decision; the rubric trace is the paper trail.

## Provenance discipline

Every piece of target work cites the LBP behavior it derives from. Every disposition decision cites its rubric trace. Non-negotiable. The target map is a regulator artifact — entries without provenance are inadmissible.

## What this skill is not

- Not a PRD writer — that is `target-map-to-prd`.
- Not a template designer — templates are the bank's responsibility.
- Not a code generator — the skill writes a map, not implementation.
- Not an architect — it fills existing template slots, it does not design new services from scratch.

## See also

- `references/disposition-rubric.md` — six-dimension rubric and decision procedure for business-rule disposition
- `references/manifest-contract.md` — the three questions every template manifest must answer
- `references/target-map-template.md` — output schema and section order for `target/<flow>-map.md`
- `../legacy-inventory/SKILL.md` — whole-app inventory (first step in the pipeline)
- `../legacy-extract-flow/SKILL.md` — per-flow deep extract (run before this skill)
- `../target-map-to-prd/SKILL.md` — turns the target map into implementation PRDs (next step)
