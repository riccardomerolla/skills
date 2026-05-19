# Legacy Modernization Skills — Design

**Date:** 2026-05-19
**Author:** Riccardo Merolla (with Claude)
**Status:** Approved for planning
**Context:** llm4zio — AI agent orchestrator for modernizing legacy J2EE/COBOL banking systems to Spring Boot microservices and a Next.js SPA.

## Purpose

Add a set of skills to this repository that let llm4zio (and a human operator) take a legacy J2EE codebase and produce, in well-defined stages, the artifacts a build team needs to modernize it onto the bank's existing production-ready templates (Next.js SPA template, Spring Boot service template, shareable-domain repos).

The skills target the *analysis and planning* half of the modernization pipeline. Implementation is done by the build team against the resulting PRDs, in the bank's templates, and is out of scope for these skills.

## Foundational decisions

The design rests on four decisions made during the brainstorm. Recording them here so they're visible to downstream readers.

### 1. Mainframe-source treatment: J2EE + targeted COBOL probes

The extract skills read the J2EE side fully (JSP, servlets, Java, ESB configuration). Mainframe code (COBOL, PL/1, JCL, copybooks) is treated as a **black box behind ESB contracts by default**. The extract skill is permitted to open specific COBOL paragraph(s) *only* when a business rule clearly lives mainframe-side and the ESB contract is not expressive enough to specify that rule — a targeted, recorded probe, not a layer.

**Rationale:** Pure black-box loses too much business logic for the modernization design to be useful. Full COBOL-as-input explodes surface area, and the majority of COBOL is CRUD glue that adds no signal. Targeted probes preserve fidelity where it matters without paying for fidelity where it doesn't.

### 2. Disposition default: strangler-fig

When mapping a flow to the target architecture, the **default disposition of every ESB-reached business rule is `stay`** — the rule remains on the mainframe and Spring Boot provides an adapter. Moves to Spring Boot are *exceptions* that must be justified per rule, with the justification recorded as part of the output.

**Rationale:** Banks modernize their J2EE front-of-house, not their mainframe systems of record. Strangler-fig matches operational reality (mainframe deploys are quarterly, regulator-reviewed, immovable), keeps regulators comfortable, and reduces risk. A non-default disposition is a deliberate engineering decision and deserves a paper trail.

The disposition decision per rule uses a stated rubric: deploy cadence, regulatory pinning, data gravity, performance envelope, blast radius, test coverage on legacy side.

### 3. Workflow shape: two-pass (shallow inventory, then per-flow deep extract)

A shallow whole-app inventory runs first, producing a system-wide map. Subsequent per-flow extracts run in any order against named flows; each extract reads the inventory so bounded contexts and shared rules are visible from slice one.

**Rationale:** Per-flow without a shared map silently mis-draws service boundaries when slice 3 reveals that slice 1's "private" rule is actually shared. Whole-app deep extract is right architecturally but ships value too late for the modernization to be economically viable. Two-pass is cheap on the shared map and incremental on the deep work.

The shallow inventory is also the natural regulator artifact ("this is the system we are modernizing"), so it earns its keep twice.

### 4. Target architecture: template manifests, not architecture docs

The mapping skill does not design Spring Boot services or Next.js structure from scratch. It reads **template manifests** declared by the bank's production-ready template repos (SPA template, Spring Boot service template, shareable-domain repos) and emits work in those terms. Each modernization project carries a config file pointing at the templates it draws from.

**Rationale:** Riccardo's setup is template-driven by design — the bank's house standards for security, ESB integration, observability, and deploy already live in the templates. Re-designing architecture per modernization project would (a) ignore those standards and (b) produce inconsistent output across projects. The template manifest is the right place for house standards to enter the skill pipeline.

## Skill set

Four new skills, in dependency order.

### Skill 1: `legacy-inventory`

**Purpose:** Shallow whole-app scan of a J2EE codebase. Produces the shared map that downstream skills read.

**Input:** path to the legacy repository.

**Output:** `legacy/inventory.md` containing —
- **JSP catalogue:** every JSP page, its location, the servlet(s) it submits to, and a one-line purpose inferred from the page title/form/headers.
- **Servlet catalogue:** every servlet, the URL pattern(s) it handles, the JSPs it dispatches to, and the ESB calls it makes.
- **ESB call surface:** every distinct ESB call observed, with inferred destination (transaction code, MQ queue, CICS program), invoking servlets, and rough payload shape.
- **Mainframe contract surface:** the set of mainframe-side endpoints reachable via ESB — transaction codes, queues, CICS programs.
- **Domain glossary:** terms observed in JSP labels, form fields, servlet names, and ESB payloads, normalized to a candidate domain vocabulary.
- **Candidate bounded contexts:** clusters of JSPs + servlets + ESB calls that share data and co-occur, proposed as future microservice boundaries. Marked as *candidate* — not final.

**Out of scope for this skill:**
- Opening COBOL or any mainframe source.
- Describing the deep behavior of any individual flow (forms, validations, business rules).
- Proposing the target architecture.

**Provenance:** every entry in the inventory cites the legacy source artifact(s) it derives from (file path + line range when applicable).

**Regulator artifact:** the inventory is the artifact handed to a compliance reviewer as "the system we are modernizing".

### Skill 2: `legacy-extract-flow`

**Purpose:** Deep per-flow extract. Produces a flow-scoped Legacy Behavior Pack entry.

**Input:** a flow name (user-supplied or proposed by the inventory's candidate bounded contexts), plus `legacy/inventory.md`.

**Output:** `legacy/flows/<flow>.md` containing —
- **Presentation surface:** every JSP in the flow with its form fields, client-side validations, navigation links, and the servlet each form posts to.
- **Controller surface:** every servlet in the flow with its request/response shape and which JSPs it dispatches to.
- **Integration surface:** every ESB call in the flow with its payload shape (inferred or extracted), trigger conditions, and downstream mainframe destination.
- **Business rules:** rules observed in the flow, each with provenance (which JSP/servlet/ESB call/COBOL paragraph it derives from) and a statement in plain language. When the rule is visible only at the mainframe side and the ESB contract is not expressive enough to specify it, the skill opens the relevant COBOL paragraph(s) — a B-mode probe — and lifts the rule, recording the COBOL location as provenance.
- **Data flow:** the shape of data as it moves from form → servlet → ESB → mainframe → back, including any transformations.
- **Edge cases:** error paths, validation failures, partial-failure handling, idempotency observed.
- **Undocumented behaviors:** anything the source does that isn't obvious from labels or comments — flag explicitly for grilling.

**Accretion rule:** subsequent runs add new flows to `legacy/flows/`. Existing flow files are not rewritten unless the user explicitly asks ("re-extract loan-origination").

**Grilling:** the output is intended to be fed to the existing `grill-with-docs` skill to surface undocumented edge cases and missing rules. No dedicated grilling skill is needed.

### Skill 3: `lbp-to-target-map`

**Purpose:** Map an extracted flow to template-shaped work in the bank's production-ready templates. Apply the strangler-fig disposition rubric to every business rule.

**Input:**
- One flow file from `legacy/flows/<flow>.md`.
- `target-templates.yaml` at the modernization-project root, declaring the template repos in scope (SPA template, Spring Boot service template, each shareable-domain repo) by path/URL and manifest location.
- Each referenced template's manifest, which declares: module structure, slot locations, naming conventions, available shareable domains, mandated libraries, security policy hooks, ESB adapter conventions, deploy model.

**Output:** `target/<flow>-map.md` containing, for each behavior in the flow:

- **SPA work:** which Next.js module/page/view files get added (concrete paths in the SPA template's conventions), which existing modules are extended, what API client surface is needed.
- **Spring Boot work:** which service is touched (existing service, new service from template, or shareable domain), what ESB adapter is added, what endpoints are exposed, what security policy attaches, what observability hooks fire.
- **Disposition per business rule** under strangler-fig default:
  - `stay` — rule remains on mainframe; Spring Boot adapter wraps it. (Default; no justification field required, but the rubric is run and recorded.)
  - `move` — rule moves to Spring Boot. **Justification required**, recorded with the rubric scores that warranted the exception.
  - `split` — rare; portion stays, portion moves. Justification and split boundary recorded.
- **Rubric trace:** for every rule, the scoring against deploy cadence, regulatory pinning, data gravity, performance envelope, blast radius, and legacy-side test coverage — even when the disposition is the default. This is the audit trail.

**Provenance:** every piece of target work cites the LBP behavior it derives from, which in turn cites legacy source.

**Regulator artifact:** the disposition decisions with their rubric traces are the regulator-readable record of "what we moved, what we kept, and why".

### Skill 4: `target-map-to-prd`

**Purpose:** Slice a target map into vertical-slice PRDs ready to feed into `to-issues`.

**Input:** `target/<flow>-map.md`.

**Output:** one or more PRD files (typically one per flow; a wide flow may produce several focused on different bounded contexts within the flow). Format follows the bank's house PRD template — the same template used by the existing `to-prd` skill — extended with:

- **Provenance section:** links back to LBP flow file and inventory entries.
- **Disposition rationale section:** the rubric traces for every business rule in the slice.
- **Template-aware Implementation Decisions:** references to specific template manifests and the slots/modules being filled.

**Distinction from `to-prd`:**
- `to-prd` takes conversation context as input; this skill takes a target map file.
- `to-prd` auto-publishes to the issue tracker; this skill writes to disk and stops, letting the user review before invoking `to-issues` for ticket creation.
- `to-prd` does not handle provenance or disposition rationale.

The skill **does not publish**. Publication is the user's call, done via the existing `to-issues` skill once the PRD has been reviewed.

## Cross-cutting: regulator artifact discipline

Every output from every skill in this set includes:

- **Provenance:** which legacy source artifact (file path + line range when applicable) each behavior, contract, or decision derives from.
- **Rationale:** when a disposition is non-default, why; even when default, the rubric trace.
- **Reviewability:** structured headings and plain-language statements suitable for a non-engineer compliance reviewer to read.

This is an output discipline applied across all four skills, not a separate skill.

## Composition

The intended workflow:

```
legacy-inventory                  → legacy/inventory.md                        (once per project)
legacy-extract-flow <flow>        → legacy/flows/<flow>.md                     (×N flows, accreting)
[grill-with-docs] <flow>          → refines legacy/flows/<flow>.md             (optional, per flow)
lbp-to-target-map <flow>          → target/<flow>-map.md                       (×N)
target-map-to-prd <flow>          → docs/prd/<flow>-*.md                       (×N)
to-issues (existing)              → tickets on tracker                         (×N)
```

Each step's output is the next step's input. Steps are idempotent at the flow level — re-running on a single flow does not disturb others.

## Template manifest schema (placeholder)

The mapping skill depends on a manifest schema declared by each template repo. The schema itself is **not specified in this design** — it should align with llm4zio's existing manifest format if one exists, or be defined by a separate proposal if not. The mapping skill's contract with the manifest is:

- The skill can ask "given a behavior of shape X, which slot in this template does it fill?"
- The skill can ask "what naming convention applies in slot Y?"
- The skill can ask "what shareable domains are available, and what behaviors do they already cover?"

If no manifest exists today, an interim convention can be a `llm4zio.manifest.yaml` at each template's root. Resolving this is a prerequisite to building Skill 3, not Skill 1 or 2.

## Skill placement in the repo

All four skills are engineering skills and live under `skills/engineering/`:

```
skills/engineering/
├── legacy-inventory/SKILL.md
├── legacy-extract-flow/SKILL.md
├── lbp-to-target-map/SKILL.md
└── target-map-to-prd/SKILL.md
```

Each must be referenced from the top-level `README.md`, added to `.claude-plugin/plugin.json`, and listed in `skills/engineering/README.md` per repo convention.

## Out of scope

- **Implementation skills.** None of these skills generate Spring Boot or Next.js code. They produce PRDs; the build team (or a separate llm4zio implementation pipeline) writes the code against the templates.
- **Manifest schema definition.** As noted above.
- **Mainframe modernization.** These skills modernize the J2EE front-of-house. COBOL/PL1/JCL on the mainframe is preserved (strangler-fig). Mainframe modernization is a separate programme outside llm4zio's current scope.
- **A dedicated legacy-grilling skill.** The existing `grill-with-docs`, pointed at a flow file, is sufficient.
- **A dedicated mainframe-boundary skill.** Boundary decisions are folded into `lbp-to-target-map` because they cannot be made without seeing the flow.
- **Legal/regulatory review of outputs.** The skills produce regulator-readable artifacts; actual sign-off is a human process.

## Open questions

1. **Manifest schema.** Does llm4zio already have a template manifest format? If yes, align to it. If no, define one before Skill 3 implementation.
2. **Flow naming and discovery.** Skill 2 takes a flow name from the user. Should the inventory's candidate bounded contexts double as a flow catalogue the user picks from, or are flows always user-named?
3. **PRD output path.** This design assumes `docs/prd/` in the modernization project. Confirm against llm4zio's existing convention.
