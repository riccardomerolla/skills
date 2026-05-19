---
name: legacy-inventory
description: Shallow whole-app scan of a legacy J2EE codebase that produces an inventory of JSPs, servlets, ESB calls, mainframe contracts, a domain glossary, and candidate bounded contexts. Use when starting a modernization analysis on a J2EE banking system, when the user wants to map a legacy app before slicing it into flows, when the user mentions "inventory the legacy app" or "scan the J2EE codebase", or as the first step in the llm4zio modernization pipeline.
---

# Legacy Inventory

Shallow whole-app scan of a J2EE codebase. No COBOL. No deep per-flow behavior. Output is `legacy/inventory.md` at the modernization project root — the shared map every downstream skill reads. This file is also the regulator artifact: "this is the system we are modernizing."

## Scope

What this skill does:

- Catalogues every JSP, servlet, ESB call, and inferred mainframe contract.
- Extracts a domain glossary from labels, form fields, servlet names, and ESB payloads.
- Proposes candidate bounded contexts by clustering co-occurring JSPs, servlets, and ESB calls.
- Attaches provenance (file path and line range) to every catalogue entry.

What this skill does NOT do:

- Open COBOL, PL/1, JCL, or any mainframe source.
- Describe the deep behavior of any individual flow — that is `legacy-extract-flow`.
- Propose a target architecture — that is `lbp-to-target-map`.
- Perform a security review or license audit.

## Process

1. **Confirm scope.** One short turn: repo path, modules to exclude, anything already known. Skip if the user already supplied it.

2. **Walk the repo.** Read in this order:
   - Top-level `README` and any docs directory.
   - Build manifests: `pom.xml`, `build.xml`, `ivy.xml`.
   - `web.xml` (servlet declarations and URL mappings).
   - All JSPs (skim titles, form actions, field names).
   - All servlet classes (URL patterns, ESB calls, JSP forwards).
   - ESB configuration: `*.bw`, `mule-config.xml`, `camel-context.xml`, any ESB-specific XML.
   - Java service classes that invoke ESB endpoints.

   Skim — read for structure, not for transcription.

3. **Build five catalogues** per the template in `references/inventory-template.md`:
   JSP catalogue, servlet catalogue, ESB call surface, mainframe contract surface, domain glossary.

4. **Propose candidate bounded contexts** by clustering servlets and JSPs that share ESB calls and form-field vocabulary. Mark every cluster **candidate** — never final.

5. **Write `legacy/inventory.md`** at the modernization project root using the template in `references/inventory-template.md`.

## Provenance discipline

Every catalogue entry must cite the source file path — and line range where applicable — it derives from. Non-negotiable. This is the regulator artifact. An entry without provenance is inadmissible.

## What this skill is not

- Not a deep extract — that is `legacy-extract-flow`.
- Not a target architecture proposal — that is `lbp-to-target-map`.
- Not a COBOL reader — mainframe source is a black box behind ESB contracts.
- Not a security review.
- Not a license audit.

## See also

- `references/inventory-template.md` — output schema and section order for `legacy/inventory.md`
- `../legacy-extract-flow/SKILL.md` — per-flow deep extract (next step)
- `../lbp-to-target-map/SKILL.md` — maps the Legacy Behavior Pack to the target architecture
- `../target-map-to-prd/SKILL.md` — turns the target map into implementation PRDs
