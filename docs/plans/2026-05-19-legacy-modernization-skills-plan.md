# Legacy Modernization Skills — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add four new engineering skills to this repository — `legacy-inventory`, `legacy-extract-flow`, `lbp-to-target-map`, `target-map-to-prd` — that, in pipeline, take a legacy J2EE/COBOL banking codebase and produce regulator-readable PRDs ready to feed into the existing `to-issues` skill.

**Architecture:** Each skill is a directory under `skills/engineering/` with a `SKILL.md` (≤ ~120 lines) and a `references/` folder for longer content. Skills compose linearly: skill 1 produces a shared map, skill 2 produces per-flow LBP entries, skill 3 maps to template-shaped target work, skill 4 emits PRDs. Strangler-fig disposition is the default; the bank's production-ready Next.js and Spring Boot templates are read via per-template manifests at the modernization project.

**Tech stack:** Markdown. The Claude plugin format (`.claude-plugin/plugin.json` skills list + per-skill `SKILL.md` with YAML frontmatter). No executable code. The reference design is committed at [docs/specs/2026-05-19-legacy-modernization-skills-design.md](../specs/2026-05-19-legacy-modernization-skills-design.md) and is the source of truth for *why* and *what*. This plan locks in *how* — exact files, exact section names, exact registry edits, in order.

---

## File structure (everything this plan creates or modifies)

**Created:**

```
skills/engineering/
├── legacy-inventory/
│   ├── SKILL.md
│   └── references/
│       └── inventory-template.md
├── legacy-extract-flow/
│   ├── SKILL.md
│   └── references/
│       ├── flow-template.md
│       └── cobol-probe-procedure.md
├── lbp-to-target-map/
│   ├── SKILL.md
│   └── references/
│       ├── disposition-rubric.md
│       ├── manifest-contract.md
│       └── target-map-template.md
└── target-map-to-prd/
    ├── SKILL.md
    └── references/
        └── prd-template.md
```

**Modified:**

- `README.md` — top-level skill index, add four new entries.
- `.claude-plugin/plugin.json` — `skills` array, add four new paths.
- `skills/engineering/README.md` — bucket index, add four new entries.

Each `SKILL.md` is kept short (≤ ~120 lines, modeled on the existing `skills/engineering/clean-room-extract/SKILL.md`). Long-form procedure detail lives in `references/`. This is the per-skill convention already established in this repo and the one mandated by `skills/productivity/write-a-skill/SKILL.md`.

**Verification approach:** This is a skills repository, not an executable codebase. There is no test runner. Each skill is verified by **trigger smoke test** — running a prompt that should activate the skill and confirming the skill's directives surface. Trigger smoke tests are bundled into Task 7. There is no TDD-style red/green loop because there is no executable artifact to assert against; the artifact is documentation that another agent reads and follows.

---

## Task 1: Repository prep

**Files:**
- Create: `skills/engineering/legacy-inventory/`
- Create: `skills/engineering/legacy-extract-flow/`
- Create: `skills/engineering/lbp-to-target-map/`
- Create: `skills/engineering/target-map-to-prd/`
- Create: `skills/engineering/legacy-inventory/references/`
- Create: `skills/engineering/legacy-extract-flow/references/`
- Create: `skills/engineering/lbp-to-target-map/references/`
- Create: `skills/engineering/target-map-to-prd/references/`

- [ ] **Step 1: Create the four skill directories with their `references/` subfolders in one command**

```bash
mkdir -p \
  skills/engineering/legacy-inventory/references \
  skills/engineering/legacy-extract-flow/references \
  skills/engineering/lbp-to-target-map/references \
  skills/engineering/target-map-to-prd/references
```

- [ ] **Step 2: Verify the directories exist**

```bash
ls -d skills/engineering/legacy-{inventory,extract-flow}/references \
      skills/engineering/{lbp-to-target-map,target-map-to-prd}/references
```

Expected: all four `references/` paths printed, no errors.

- [ ] **Step 3: No commit yet**

Empty directories are not tracked by git. Task 2 produces the first committable artifact.

---

## Task 2: `legacy-inventory` skill

**Files:**
- Create: `skills/engineering/legacy-inventory/SKILL.md`
- Create: `skills/engineering/legacy-inventory/references/inventory-template.md`

### Skill 2.1: Write `SKILL.md`

- [ ] **Step 1: Create `skills/engineering/legacy-inventory/SKILL.md` with this structure**

The file MUST contain (in this order):

**Frontmatter:**
```yaml
---
name: legacy-inventory
description: Shallow whole-app scan of a legacy J2EE codebase that produces an inventory of JSPs, servlets, ESB calls, mainframe contracts, a domain glossary, and candidate bounded contexts. Use when starting a modernization analysis on a J2EE banking system, when the user wants to map a legacy app before slicing it into flows, when the user mentions "inventory the legacy app" or "scan the J2EE codebase", or as the first step in the llm4zio modernization pipeline.
---
```

**Body sections (H2 headings, in this order):**

1. `# Legacy Inventory` — one paragraph stating: shallow whole-app scan, no COBOL, no deep behavior, output is `legacy/inventory.md`, this is also the regulator artifact for "the system we are modernizing".

2. `## Scope` — bullet list of what the skill does and explicitly does NOT do. Mirror the "Out of scope for this skill" section from the design doc (no COBOL, no per-flow deep behavior, no target architecture proposal).

3. `## Process` — numbered steps:
   1. Confirm scope with user in one short turn: repo path, anything to exclude. Skip if already supplied.
   2. Walk the repo. Read in this order: top-level `README` and docs, `pom.xml`/`build.xml`/`ivy.xml`, `web.xml`, JSPs, servlets, ESB configuration (typical files: `*.bw`, `mule-config.xml`, `camel-context.xml`, ESB-specific XML), Java service classes that call ESB endpoints. Skim — do not transcribe.
   3. Build five catalogues per the template in `references/inventory-template.md`: JSP catalogue, servlet catalogue, ESB call surface, mainframe contract surface, domain glossary.
   4. Propose candidate bounded contexts by clustering servlets and JSPs that share ESB calls and form-field vocabulary. Mark them **candidate** — never final.
   5. Write `legacy/inventory.md` at the modernization project's root using the template.

4. `## Provenance discipline` — every catalogue entry must cite the source file path (and line range when applicable) it derives from. Non-negotiable; this is the regulator artifact.

5. `## What this skill is not` — bullets: not a deep extract (that's `legacy-extract-flow`), not a target architecture proposal (that's `lbp-to-target-map`), not a COBOL reader, not a security review, not a license audit.

6. `## See also` — link to `references/inventory-template.md`, and link to the sibling skills (`legacy-extract-flow`, `lbp-to-target-map`, `target-map-to-prd`).

Target file length: ≤ 100 lines. If approaching the limit, push narrative into `references/` rather than inflating `SKILL.md`.

- [ ] **Step 2: Verify line count**

```bash
wc -l skills/engineering/legacy-inventory/SKILL.md
```

Expected: a number ≤ 100. If over, move detail into `references/inventory-template.md`.

### Skill 2.2: Write `references/inventory-template.md`

- [ ] **Step 3: Create `skills/engineering/legacy-inventory/references/inventory-template.md`**

This file is the *output template* the skill writes into. It MUST contain:

- A top-of-file note: "This is the template for the inventory output. The skill writes a populated copy to `legacy/inventory.md` at the modernization project root. Section order is mandatory; section headings are mandatory; bullet structure within each section is recommended."

- Eight numbered sections, each with a one-line purpose and an example row:

  1. **Overview** — system name, source repo path, date of scan, scan scope (repo, modules, excluded paths), one-paragraph system summary.
  2. **JSP catalogue** — table with columns: `JSP path | one-line purpose | form posts to (servlet) | provenance (file:line)`.
  3. **Servlet catalogue** — table with columns: `servlet class | URL pattern(s) | JSPs dispatched to | ESB calls invoked | provenance`.
  4. **ESB call surface** — table with columns: `call ID | destination kind (transaction code / MQ queue / CICS program / other) | destination name | invoking servlets | rough payload shape | provenance`.
  5. **Mainframe contract surface** — table with columns: `contract ID | transaction code or program | direction (in / out / both) | invoked from (ESB call IDs) | known semantics (one line if visible from ESB; otherwise "opaque") | provenance`.
  6. **Domain glossary** — table with columns: `term | observed in (JSP labels / form fields / servlet names / ESB payloads) | candidate canonical name | notes`.
  7. **Candidate bounded contexts** — for each candidate: name, included JSPs, included servlets, included ESB calls, included contract IDs, one-paragraph rationale.
  8. **Open questions for the user** — anything ambiguous the scan turned up: orphan JSPs, servlets with no callers, ESB calls with no invokers, terms with conflicting meanings.

- A closing reminder: every entry must carry provenance; the inventory is a regulator artifact.

- [ ] **Step 4: Commit Task 2**

```bash
git add skills/engineering/legacy-inventory/
git commit -m "$(cat <<'EOF'
Add legacy-inventory skill

Shallow whole-app scan of a J2EE codebase producing the shared
inventory map (JSPs, servlets, ESB calls, mainframe contracts,
domain glossary, candidate bounded contexts) that the rest of
the legacy-modernization skill pipeline reads.
EOF
)"
```

---

## Task 3: `legacy-extract-flow` skill

**Files:**
- Create: `skills/engineering/legacy-extract-flow/SKILL.md`
- Create: `skills/engineering/legacy-extract-flow/references/flow-template.md`
- Create: `skills/engineering/legacy-extract-flow/references/cobol-probe-procedure.md`

### Skill 3.1: Write `SKILL.md`

- [ ] **Step 1: Create `skills/engineering/legacy-extract-flow/SKILL.md` with this structure**

**Frontmatter:**
```yaml
---
name: legacy-extract-flow
description: Deep extract of a single named flow from a legacy J2EE app — JSP forms, servlet routing, ESB calls with payload shapes, business rules with provenance, edge cases, undocumented behaviors. Permitted to perform targeted COBOL probes when a business rule lives mainframe-side and the ESB contract does not specify it. Reads the inventory produced by legacy-inventory. Use when extracting a specific flow ("extract loan-origination", "deep dive on this JSP flow") for modernization, after running legacy-inventory.
---
```

**Body sections (H2 headings, in this order):**

1. `# Legacy Extract Flow` — one paragraph: deep per-flow extract, reads `legacy/inventory.md`, output accretes at `legacy/flows/<flow>.md`, never rewrites an existing flow file unless explicitly asked.

2. `## Preconditions` — `legacy/inventory.md` must exist (instruct the user to run `legacy-inventory` first if not); a flow name must be supplied (or chosen from the inventory's candidate bounded contexts).

3. `## Process` — numbered steps:
   1. Read `legacy/inventory.md`. Identify the subset relevant to the named flow: JSPs, servlets, ESB calls, contract IDs.
   2. Read those JSP and servlet sources in full. Extract presentation surface (forms, validations, navigation), controller surface (routing, request/response shapes, dispatch).
   3. For each ESB call in the flow, extract the inferred payload shape and the trigger conditions in the servlet.
   4. Enumerate the business rules the flow encodes. For each rule, attempt to specify it from JSP + servlet + ESB contract alone. If a rule clearly lives mainframe-side and the ESB contract does not expose enough to specify it, invoke the **B-mode COBOL probe** — see `references/cobol-probe-procedure.md`. Lift the rule and record COBOL location as provenance.
   5. Capture data flow end-to-end: form → servlet → ESB → mainframe → return path.
   6. Capture edge cases: error paths, validation failures, partial-failure handling, idempotency.
   7. Flag undocumented behaviors explicitly for downstream grilling.
   8. Write the flow file using `references/flow-template.md`. If the file already exists, refuse and ask the user whether to overwrite.

4. `## The COBOL probe boundary` — a short paragraph reinforcing: probes are surgical, recorded, and only invoked when ESB contracts are insufficient. Probing is never the *default* path. See the procedure in `references/cobol-probe-procedure.md`.

5. `## Provenance discipline` — every behavior, rule, and contract must cite source (file path + line range). For COBOL-probed rules, both the ESB-side trigger and the COBOL paragraph are recorded.

6. `## Accretion rule` — subsequent runs add new flows under `legacy/flows/`. They do not modify existing flow files. To refresh an existing flow file, the user must explicitly say so.

7. `## Grilling` — after writing a flow file, suggest the user runs `grill-with-docs` pointed at the new flow file to surface missing rules and undocumented edge cases. Do not invoke `grill-with-docs` automatically.

8. `## What this skill is not` — not an inventory (that's `legacy-inventory`), not a target architecture mapping (that's `lbp-to-target-map`), not a wholesale COBOL reader, not a security review.

9. `## See also` — link to `references/flow-template.md`, `references/cobol-probe-procedure.md`, and sibling skills.

Target length: ≤ 120 lines.

- [ ] **Step 2: Verify line count**

```bash
wc -l skills/engineering/legacy-extract-flow/SKILL.md
```

Expected: ≤ 120. If over, push material into `references/`.

### Skill 3.2: Write `references/flow-template.md`

- [ ] **Step 3: Create `skills/engineering/legacy-extract-flow/references/flow-template.md`**

Output template for `legacy/flows/<flow>.md`. MUST contain:

- Top-of-file note explaining: the skill writes one populated copy per flow; section order is mandatory; provenance fields are mandatory.

- Sections, in order:

  1. **Flow header** — flow name, extraction date, source repo, inventory file pointer, scan scope (which JSPs, servlets, ESB calls were considered in-scope for this flow).
  2. **Presentation surface** — for each JSP: path, purpose, form fields (name, type, client-side validations), navigation links, posts-to servlet, provenance.
  3. **Controller surface** — for each servlet: class, URL pattern, request shape, response shape, JSPs dispatched, ESB calls invoked, provenance.
  4. **Integration surface** — for each ESB call: call ID (matches inventory), inferred payload shape (request + response), trigger conditions, destination contract ID, provenance.
  5. **Business rules** — for each rule: one-sentence rule statement in plain language, provenance (the JSP / servlet / ESB call / COBOL paragraph the rule derives from), trigger conditions, observed examples if any.
  6. **Data flow** — narrative of the shape of data as it moves through the flow, with one diagram-style ASCII sketch or numbered sequence (form fields → servlet variables → ESB request payload → mainframe contract → response payload → servlet response → JSP render).
  7. **Edge cases** — error paths, validation failures, partial-failure handling, idempotency observed.
  8. **Undocumented behaviors** — explicit list of things the source does that aren't obvious from labels, comments, or contracts. Each item is a candidate for grilling.
  9. **Open questions** — anything ambiguous the extract turned up.

### Skill 3.3: Write `references/cobol-probe-procedure.md`

- [ ] **Step 4: Create `skills/engineering/legacy-extract-flow/references/cobol-probe-procedure.md`**

This file defines the **B-mode COBOL probe** as a procedure. MUST contain:

- **Preamble:** probes are surgical, recorded, and exceptional. The default extraction path reads only J2EE source. A probe is only justified when (a) a business rule clearly lives mainframe-side, AND (b) the ESB contract for the relevant call is not expressive enough to specify the rule for a Spring Boot adapter to wrap correctly.

- **Procedure (numbered steps):**
  1. **Justify the probe.** Write one sentence stating why the ESB contract is insufficient and what behavior is needed. Refuse to probe without this sentence in the flow file's business-rules section.
  2. **Locate the paragraph(s).** Map the ESB call's transaction code or program name to the COBOL program. Use the mainframe contract surface in the inventory as the starting point. Read the program's PROCEDURE DIVISION; locate the paragraph(s) that implement the rule.
  3. **Lift the rule, not the code.** Restate the rule in plain language in the flow file's business-rules section. Do not transcribe COBOL syntax. Cite the COBOL location (program + paragraph + line range) as provenance.
  4. **Stop at the rule boundary.** Do not extract surrounding glue (file I/O, DB calls, formatting). The probe ends when the rule is captured.
  5. **Record the probe in an audit log section at the bottom of the flow file** — every probe is listed: ESB call ID, COBOL location, rule extracted, justification. This is the regulator-facing audit trail of mainframe access.

- **Forbidden:** opening COBOL for "context", "completeness", or because the J2EE-side rule was easy enough to specify without it. Defaulting to probes destroys the strangler-fig posture and pollutes the regulator artifact with unnecessary mainframe reads.

- [ ] **Step 5: Commit Task 3**

```bash
git add skills/engineering/legacy-extract-flow/
git commit -m "$(cat <<'EOF'
Add legacy-extract-flow skill

Per-flow deep extract of a J2EE flow's presentation, controller,
integration, and business-rule surfaces, with a targeted B-mode
COBOL probe procedure for rules that cannot be specified from
the ESB contract alone. Accretes into legacy/flows/.
EOF
)"
```

---

## Task 4: `lbp-to-target-map` skill

**Files:**
- Create: `skills/engineering/lbp-to-target-map/SKILL.md`
- Create: `skills/engineering/lbp-to-target-map/references/disposition-rubric.md`
- Create: `skills/engineering/lbp-to-target-map/references/manifest-contract.md`
- Create: `skills/engineering/lbp-to-target-map/references/target-map-template.md`

### Skill 4.1: Write `SKILL.md`

- [ ] **Step 1: Create `skills/engineering/lbp-to-target-map/SKILL.md` with this structure**

**Frontmatter:**
```yaml
---
name: lbp-to-target-map
description: Map an extracted legacy flow to template-shaped modernization work in the bank's production-ready Next.js SPA template and Spring Boot service template, with strangler-fig disposition by default for every business rule. Reads a flow file from legacy-extract-flow plus template manifests declared at the modernization project root. Use when the user wants to design the target work for a specific extracted flow, or says "map this flow to the templates", "apply strangler-fig to this flow", or "what should we build for loan-origination".
---
```

**Body sections (H2 headings, in this order):**

1. `# LBP to Target Map` — one paragraph: maps a flow to template-shaped work; strangler-fig is the default disposition for every business rule; reads template manifests; output is `target/<flow>-map.md`.

2. `## Preconditions` — `legacy/flows/<flow>.md` exists; `target-templates.yaml` (or equivalent) at modernization project root declares: SPA template manifest path, Spring Boot service template manifest path, each shareable-domain manifest path.

3. `## Manifest contract` — the skill expects each referenced manifest to answer three questions: (a) given a behavior of shape X, which slot in this template fills it; (b) what naming convention applies in slot Y; (c) what shareable domains are available and what behaviors they already cover. The full contract lives in `references/manifest-contract.md`. If a referenced manifest cannot answer all three for a given behavior, the skill flags the gap and stops for that behavior rather than guessing.

4. `## Process` — numbered steps:
   1. Read the flow file. Read the inventory file for cross-references. Read `target-templates.yaml` and the manifest files it points at.
   2. For each entry in the flow's presentation surface, map to SPA work using the SPA template manifest: concrete module/page/view paths to create, existing modules to extend, API client surface needed.
   3. For each entry in the flow's controller and integration surfaces, map to Spring Boot work using the service-template and shareable-domain manifests: target service (existing, new from template, or shareable domain), ESB adapter additions, endpoints to expose, security policy hooks, observability hooks.
   4. For **every** business rule in the flow, run the disposition rubric in `references/disposition-rubric.md`. Default disposition is `stay`. Record the rubric trace (one line per rubric dimension with a brief reason) regardless of disposition. Non-`stay` dispositions REQUIRE an explicit justification field.
   5. Write `target/<flow>-map.md` using `references/target-map-template.md`.

5. `## Strangler-fig default` — short paragraph: every ESB-reached business rule is `stay` by default; `move` and `split` are exceptions that require justification; the rubric is run for every rule including `stay` ones so the audit trail is uniform.

6. `## Provenance discipline` — every piece of target work cites the LBP behavior it derives from. Every disposition decision cites its rubric trace.

7. `## What this skill is not` — not a PRD writer (that's `target-map-to-prd`), not a template designer, not a code generator. The skill writes a map, not implementation.

8. `## See also` — link to `references/disposition-rubric.md`, `references/manifest-contract.md`, `references/target-map-template.md`, and sibling skills.

Target length: ≤ 120 lines.

- [ ] **Step 2: Verify line count**

```bash
wc -l skills/engineering/lbp-to-target-map/SKILL.md
```

Expected: ≤ 120.

### Skill 4.2: Write `references/disposition-rubric.md`

- [ ] **Step 3: Create `skills/engineering/lbp-to-target-map/references/disposition-rubric.md`**

MUST contain:

- **Preamble.** The rubric is applied to every business rule in a flow. The default disposition is `stay` (mainframe + Spring Boot adapter). `move` (rule migrates to Spring Boot) and `split` (rule partially moves) are exceptions. The rubric trace is recorded for every rule including default-`stay` rules — this uniform audit trail is the regulator artifact for the boundary decision.

- **The six rubric dimensions.** Each has a question, the signal that biases toward `stay`, and the signal that biases toward `move`:

  1. **Deploy cadence** — How often does this rule change?
     - `stay` signal: rare changes or aligned with the mainframe quarterly release.
     - `move` signal: monthly or more frequent business-driven changes that the mainframe release calendar cannot accommodate.

  2. **Regulatory pinning** — Is the rule pinned by regulation or audit?
     - `stay` signal: rule cited in regulator reviews, present in audit-controlled COBOL, central to the mainframe system of record.
     - `move` signal: rule is purely UX or workflow with no regulatory pinning.

  3. **Data gravity** — Where does the data the rule operates on live?
     - `stay` signal: rule depends on mainframe-resident data that would require expensive replication.
     - `move` signal: rule operates on data already available outside the mainframe, or on data that is naturally Spring-Boot-side (session, request, derived).

  4. **Performance envelope** — Does the rule need mainframe-class performance?
     - `stay` signal: high-volume, low-latency, batch-coupled, or transactionally coupled to mainframe operations.
     - `move` signal: workflow-paced, user-facing, or low-volume.

  5. **Blast radius** — What happens when this rule breaks?
     - `stay` signal: incorrect behavior affects regulated outputs, system-of-record entries, or cross-system integrity.
     - `move` signal: incorrect behavior is recoverable, user-visible-only, or contained to one service.

  6. **Legacy-side test coverage** — How well-tested is the existing rule?
     - `stay` signal: rule is heavily tested on the mainframe side; moving forfeits that coverage.
     - `move` signal: rule has weak or no mainframe tests, OR the modernization will add stronger tests Spring-Boot-side.

- **Decision procedure:**
  1. Score each dimension as `stay-leaning`, `neutral`, or `move-leaning` with a one-line reason citing the flow file.
  2. If all six are `stay-leaning` or `neutral`: disposition is `stay`. No justification required beyond the trace.
  3. If three or more are `move-leaning`: disposition is `move`. Justification required: name the dimensions that drove the exception.
  4. If one or two are `move-leaning` and the rest are mixed: this is a `split` candidate. Justification required: state explicitly which portion moves and which stays, and why splitting is preferable to a clean `stay` adapter.
  5. The rubric trace and disposition are written into the target map's business-rules section. Always.

- **Examples** (two or three): a `stay` example (e.g. "interest accrual rule — all six stay-leaning"), a `move` example (e.g. "promo eligibility rule — frequent change + no regulatory pinning + weak mainframe tests"), and a `split` example.

### Skill 4.3: Write `references/manifest-contract.md`

- [ ] **Step 4: Create `skills/engineering/lbp-to-target-map/references/manifest-contract.md`**

MUST contain:

- **Preamble.** This file specifies the contract between `lbp-to-target-map` and the template manifests it reads. The contract is **not** a schema specification — that lives with llm4zio. This file specifies what the skill *asks* of any manifest. A manifest that satisfies the three questions, in any format, is sufficient.

- **The three questions every manifest must answer:**

  1. **Slot resolution** — *given a behavior of shape X, which slot in this template fills it?*
     - For the SPA manifest: shapes include "user-facing form", "list view", "detail view", "wizard step", "modal", "API client call". Slot must resolve to a concrete module path and the file conventions for adding new content (file naming, route registration, state management hooks).
     - For the Spring Boot service-template manifest: shapes include "REST endpoint", "ESB adapter", "domain service", "security policy", "observability hook", "scheduled job". Slot must resolve to package path and file conventions.
     - For a shareable-domain manifest: shapes include "existing capability we can reuse". Slot must resolve to the public API of the shared domain and how to import/depend on it.

  2. **Convention surfacing** — *what naming convention, file structure, and base-class/interface conventions apply in slot Y?*
     - The skill writes target maps that reference concrete file paths and types. Conventions must be machine-readable enough that the skill writes correct paths and names rather than placeholders.

  3. **Capability inventory** — *what shareable domains are available, and what behaviors do they already cover?*
     - For every shareable-domain manifest listed in `target-templates.yaml`, the skill must be able to enumerate covered capabilities. A behavior in the flow that matches an existing capability should be mapped to that capability, not to a new implementation.

- **Behavior when the contract is unmet:**
  - If a manifest cannot answer a question for a specific behavior, the skill records the gap in the target map's open-questions section and emits a recommendation for the manifest authors to extend the manifest. The skill does not invent missing slots or guess.

- **Note for llm4zio integration:** if llm4zio already has a manifest schema, the skill should align field names with it. If not, the skill treats the three questions as the canonical contract and surfaces ambiguity to the user.

### Skill 4.4: Write `references/target-map-template.md`

- [ ] **Step 5: Create `skills/engineering/lbp-to-target-map/references/target-map-template.md`**

Output template for `target/<flow>-map.md`. MUST contain:

- Top-of-file note: the skill writes one populated copy per flow.

- Sections, in order:

  1. **Map header** — flow name, source flow file pointer, target-templates.yaml pointer, list of manifests consulted, date, summary disposition (counts of `stay` / `move` / `split` rules).
  2. **SPA work** — for each presentation-surface entry in the flow file: source LBP entry pointer, target template + slot, concrete file additions (paths in the SPA template's conventions), existing modules extended, API client surface needed, manifest citation.
  3. **Spring Boot work** — for each controller/integration entry: source LBP entry pointer, target service (existing service / new from template / shareable domain by name), ESB adapter additions, endpoints exposed, security policy hooks, observability hooks, manifest citation.
  4. **Business rules and dispositions** — for each rule: rule statement (copied from flow file), source provenance (copied from flow file), full rubric trace (one line per dimension), disposition (`stay` / `move` / `split`), justification when non-`stay`, target work that implements the disposition (Spring Boot adapter for `stay`, service code for `move`, both for `split`).
  5. **Cross-cutting** — anything that touches multiple slices: shared types, shared security policies, shared ESB adapters used by multiple rules.
  6. **Open questions** — manifest gaps (per the manifest contract), flow-file ambiguities surfaced during mapping, decisions deferred to the user.

- [ ] **Step 6: Commit Task 4**

```bash
git add skills/engineering/lbp-to-target-map/
git commit -m "$(cat <<'EOF'
Add lbp-to-target-map skill

Maps an extracted legacy flow to template-shaped modernization
work against the bank's production-ready Next.js SPA and Spring
Boot service templates. Strangler-fig disposition by default for
every business rule, with a six-dimension rubric whose trace is
recorded uniformly as the regulator-facing audit trail.
EOF
)"
```

---

## Task 5: `target-map-to-prd` skill

**Files:**
- Create: `skills/engineering/target-map-to-prd/SKILL.md`
- Create: `skills/engineering/target-map-to-prd/references/prd-template.md`

### Skill 5.1: Write `SKILL.md`

- [ ] **Step 1: Create `skills/engineering/target-map-to-prd/SKILL.md` with this structure**

**Frontmatter:**
```yaml
---
name: target-map-to-prd
description: Slice a target map produced by lbp-to-target-map into vertical-slice PRDs in the bank's house PRD template, extended with provenance and strangler-fig disposition rationale, ready to feed into the existing to-issues skill. Use when the user wants to convert a target map to PRDs, slice modernization work into independently-grabbable tickets, or says "PRDs from this map", "ticket slices for loan-origination", or "produce the build-team handoff for this flow".
---
```

**Body sections (H2 headings, in this order):**

1. `# Target Map to PRD` — one paragraph: slices a target map into vertical-slice PRDs; format follows the bank's house template (the same one `to-prd` uses) extended with provenance and disposition fields; writes to disk and stops — does not auto-publish; user invokes `to-issues` afterward.

2. `## Preconditions` — `target/<flow>-map.md` exists. The bank's house PRD template (as encoded in `references/prd-template.md`) is the format target.

3. `## Process` — numbered steps:
   1. Read the target map file. Identify natural vertical slices: typically one slice per flow, but a wide flow with multiple bounded contexts produces multiple slices.
   2. For each slice, draft a PRD using `references/prd-template.md`. Fill: problem statement (from the LBP flow's purpose), solution (from the target map's SPA + Spring Boot work), user stories (one per presentation-surface entry, expanded with edge cases from the LBP), implementation decisions (specific template slots, services, ESB adapters), testing decisions (informed by the LBP's edge cases and undocumented behaviors), out of scope, provenance section, disposition rationale section.
   3. Write each PRD to `docs/prd/<flow>-<slice-name>.md` at the modernization project root.
   4. Do **not** publish. Tell the user the PRDs are written; suggest reviewing them and then invoking `to-issues` for ticket creation.

4. `## Distinction from `to-prd`** — bullets:
   - `to-prd` takes conversation context; this skill takes a target map file.
   - `to-prd` auto-publishes to the issue tracker; this skill writes to disk and stops.
   - `to-prd` does not include provenance or disposition rationale; this skill does.

5. `## Provenance discipline` — every PRD includes a provenance section pointing back to the target map, the source LBP flow file, and the inventory. Every disposition rationale section copies the rubric traces from the target map. These two sections are non-negotiable; they're the regulator-readable audit trail.

6. `## What this skill is not` — not a publisher (use `to-issues`), not an interviewer (use `to-prd` for conversation-context-driven PRDs), not a code generator.

7. `## See also` — link to `references/prd-template.md`, the existing `to-issues` and `to-prd` skills, and sibling skills.

Target length: ≤ 100 lines.

- [ ] **Step 2: Verify line count**

```bash
wc -l skills/engineering/target-map-to-prd/SKILL.md
```

Expected: ≤ 100.

### Skill 5.2: Write `references/prd-template.md`

- [ ] **Step 3: Create `skills/engineering/target-map-to-prd/references/prd-template.md`**

This file is the bank's house PRD template extended with provenance and disposition fields. MUST contain:

- Top-of-file note: extends the `to-prd` template; field order is mandatory; the **Provenance** and **Disposition Rationale** sections are mandatory and may not be omitted.

- Sections, in order:

  1. **Problem Statement** — from the user's perspective. One-paragraph.
  2. **Solution** — from the user's perspective. One-paragraph.
  3. **User Stories** — long numbered list. Format: `As an <actor>, I want a <feature>, so that <benefit>`. Cover every presentation-surface entry and every edge case from the source LBP.
  4. **Implementation Decisions** — concrete template slots, services, ESB adapters, shareable domains used. References specific manifest paths. No code snippets, no file paths likely to change.
  5. **Testing Decisions** — what to test (external behavior), modules to test, prior art for similar tests. Informed by the LBP's edge cases and undocumented behaviors.
  6. **Out of Scope** — what this PRD does NOT cover. Particularly important: which other slices in the same flow are deferred to other PRDs.
  7. **Provenance** — pointers to: the target map file, the source LBP flow file, the inventory file, the relevant manifest files. This section is mandatory.
  8. **Disposition Rationale** — for every business rule covered by this PRD: rule statement, rubric trace, disposition, justification (if non-`stay`). This section is mandatory and is the regulator-facing audit trail.
  9. **Further Notes** — anything else.

- [ ] **Step 4: Commit Task 5**

```bash
git add skills/engineering/target-map-to-prd/
git commit -m "$(cat <<'EOF'
Add target-map-to-prd skill

Slices a target map into vertical-slice PRDs in the bank's house
template, extended with provenance and strangler-fig disposition
rationale sections. Writes to disk; does not auto-publish — the
existing to-issues skill handles ticket creation afterward.
EOF
)"
```

---

## Task 6: Registry updates

**Files:**
- Modify: `README.md`
- Modify: `.claude-plugin/plugin.json`
- Modify: `skills/engineering/README.md`

### Skill 6.1: Update `.claude-plugin/plugin.json`

- [ ] **Step 1: Read current `.claude-plugin/plugin.json`**

```bash
cat .claude-plugin/plugin.json
```

The file is a JSON object with a `skills` array. The array currently ends with `./skills/productivity/write-a-skill`.

- [ ] **Step 2: Add the four new skill paths to the `skills` array**

Add these four entries in alphabetical order within the `engineering/` group (which begins at `./skills/engineering/clean-room-extract`):

```
"./skills/engineering/lbp-to-target-map",
"./skills/engineering/legacy-extract-flow",
"./skills/engineering/legacy-inventory",
"./skills/engineering/target-map-to-prd",
```

Specifically: insert these so they slot alphabetically among the existing engineering entries. After this change, the engineering block of the `skills` array contains, in order: `clean-room-extract`, `csp-to-prd`, `diagnose`, `grill-with-docs`, `improve-codebase-architecture`, `lbp-to-target-map`, `legacy-extract-flow`, `legacy-inventory`, `scala3-zio`, `setup-ricky-skills`, `target-map-to-prd`, `tdd`, `to-issues`, `to-prd`, `triage`, `zen-of-ricky`, `zoom-out`.

Note: the existing file does not have a strictly alphabetical ordering today (`triage` and `improve-codebase-architecture` are out of order). Match the existing convention rather than imposing a new one — insert each new entry near alphabetically-related neighbors.

- [ ] **Step 3: Validate the JSON parses**

```bash
python3 -c "import json; json.load(open('.claude-plugin/plugin.json'))"
```

Expected: no output, exit code 0.

### Skill 6.2: Update top-level `README.md`

- [ ] **Step 4: Read current top-level `README.md`**

```bash
cat README.md
```

Note its structure — confirm it lists engineering skills with names linked to their `SKILL.md`.

- [ ] **Step 5: Add four entries to the engineering section of `README.md`**

For each new skill, add a one-line entry of the form:

```
- [legacy-inventory](skills/engineering/legacy-inventory/SKILL.md) — shallow whole-app scan of a legacy J2EE codebase producing the shared inventory map (JSPs, servlets, ESB calls, mainframe contracts, candidate bounded contexts).
- [legacy-extract-flow](skills/engineering/legacy-extract-flow/SKILL.md) — deep per-flow extract with a targeted COBOL probe for rules unreachable from the ESB contract alone.
- [lbp-to-target-map](skills/engineering/lbp-to-target-map/SKILL.md) — maps an extracted flow to template-shaped work in the bank's Next.js and Spring Boot templates, strangler-fig by default with a rubric-traced disposition per business rule.
- [target-map-to-prd](skills/engineering/target-map-to-prd/SKILL.md) — slices a target map into vertical-slice PRDs in the bank's PRD template, extended with provenance and disposition rationale.
```

Insert these in the engineering section, near alphabetically-related neighbors, matching the existing README ordering.

### Skill 6.3: Update `skills/engineering/README.md`

- [ ] **Step 6: Read current `skills/engineering/README.md`**

```bash
cat skills/engineering/README.md
```

- [ ] **Step 7: Add four entries to the bucket README**

Same four entries as the top-level README, formatted to match the bucket README's existing convention. The bucket README's link href is relative to its own location, so the links shorten to `legacy-inventory/SKILL.md`, etc.:

```
- [legacy-inventory](legacy-inventory/SKILL.md) — shallow whole-app scan of a legacy J2EE codebase producing the shared inventory map.
- [legacy-extract-flow](legacy-extract-flow/SKILL.md) — deep per-flow extract with a targeted COBOL probe.
- [lbp-to-target-map](lbp-to-target-map/SKILL.md) — maps an extracted flow to template-shaped work, strangler-fig by default.
- [target-map-to-prd](target-map-to-prd/SKILL.md) — slices a target map into vertical-slice PRDs with provenance and disposition rationale.
```

- [ ] **Step 8: Verify the links resolve**

```bash
ls skills/engineering/legacy-inventory/SKILL.md \
   skills/engineering/legacy-extract-flow/SKILL.md \
   skills/engineering/lbp-to-target-map/SKILL.md \
   skills/engineering/target-map-to-prd/SKILL.md
```

Expected: all four files listed, no errors.

- [ ] **Step 9: Commit Task 6**

```bash
git add README.md .claude-plugin/plugin.json skills/engineering/README.md
git commit -m "$(cat <<'EOF'
Register legacy-modernization skills

Add the four legacy-modernization skills to the top-level README,
the engineering bucket README, and .claude-plugin/plugin.json so
they're discoverable and installable from the plugin.
EOF
)"
```

---

## Task 7: Trigger smoke tests

Skills are verified by trigger smoke test, not unit test. Each smoke test is a prompt that should activate the skill. The check is: when the prompt is run, does Claude announce it is using the skill?

These tests are intended to be run by the user (or by Claude in a fresh session) after the skills are installed. The expected outcome is documented inline so a reviewer can confirm.

- [ ] **Step 1: Smoke test `legacy-inventory`**

Run prompt: *"I want to start a modernization analysis on a J2EE banking app at `~/work/oldbank/`. Can you inventory it?"*

Expected: Claude invokes the `legacy-inventory` skill (announces "Using legacy-inventory to ...").

- [ ] **Step 2: Smoke test `legacy-extract-flow`**

Run prompt: *"The inventory says there's a loan-origination flow. Extract it."*

Expected: Claude invokes `legacy-extract-flow`. (If the inventory file doesn't exist, the skill should refuse and ask the user to run `legacy-inventory` first — that refusal is correct behavior.)

- [ ] **Step 3: Smoke test `lbp-to-target-map`**

Run prompt: *"Map the loan-origination flow to our templates."*

Expected: Claude invokes `lbp-to-target-map`.

- [ ] **Step 4: Smoke test `target-map-to-prd`**

Run prompt: *"Produce the PRDs from the loan-origination target map."*

Expected: Claude invokes `target-map-to-prd`.

- [ ] **Step 5: Verify description triggers don't collide**

Run prompt: *"Reverse-engineer this repo to a clean-room spec."*

Expected: Claude invokes `clean-room-extract`, not any of the new legacy-modernization skills. (This test guards against description-string collision — the legacy-modernization skills' descriptions reference J2EE / banking modernization specifically; the clean-room skill should still win for license-clean reimplementation prompts.)

- [ ] **Step 6: Record smoke-test results**

If any smoke test fails (wrong skill activates, or no skill activates), the root cause is almost always the description string in the failing skill's frontmatter. Tighten the description, commit the fix, re-test.

No commit at this task unless smoke tests required fixes.

---

## Self-review notes

The plan was reviewed for the patterns called out in the writing-plans skill:

- **Spec coverage:** Every section of the design doc is covered. Foundational decisions 1–4 are encoded in the relevant skills' procedures and references. The four-skill set in the design maps 1:1 to Tasks 2–5. Cross-cutting regulator-artifact discipline is encoded in each `SKILL.md`'s `## Provenance discipline` section (mandatory per the plan). The open questions from the design (manifest schema, flow naming, PRD output path) are surfaced as: manifest contract spec in Skill 4.3, flow naming flexibility in Skill 3's process step 2, PRD output path locked to `docs/prd/<flow>-<slice>.md` in Skill 5.1's process step 3.

- **Placeholder scan:** No "TBD" or "implement later" tokens. The `references/manifest-contract.md` spec deliberately does not specify a schema — it specifies a *contract* (three questions any manifest must answer) and explicitly defers schema choice to llm4zio. That is a design decision, not a placeholder.

- **Type consistency:** File paths and skill names are consistent across tasks (`legacy/inventory.md`, `legacy/flows/<flow>.md`, `target/<flow>-map.md`, `docs/prd/<flow>-<slice>.md`). Reference file names match between SKILL.md sections and the reference-file-creation steps.

- **TDD adaptation:** This repo has no test runner; the artifacts are documentation other agents read. The TDD red/green/refactor cycle from the writing-plans default does not apply. Task 7's trigger smoke tests are the closest analogue and are explicit about what verifies each skill.
