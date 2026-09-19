---
name: csp-to-prd
description: Split a Clean Specification Pack (CSP) into a set of independently-grabbable PRDs that the build team uses to reimplement the system from scratch. Continuation of the clean-room-extract workflow: operates on the `csp/` folder, never on the original source. Use whenever the user wants to turn a CSP into PRDs, slice a spec into build tickets, plan the build phase of a clean-room operation, generate work items from a CSP, or says things like "csp to prd", "split the spec into PRDs", "generate PRDs from the CSP", "to-spec from CSP", or asks how to hand the CSP off to engineers.
---

# CSP → PRDs

Take a Clean Specification Pack (the `csp/` folder produced by `clean-room-extract`) as input. Produce a `prds/` folder of independently-grabbable PRDs as output, each describing one slice of the build the build team must implement.

You are still inside the clean-room boundary. The PRDs are written **for the build team**, who has never seen the original source. That means:

- The CSP is the only source of truth. PRDs derive from it and cite it; they MUST NOT introduce facts the CSP doesn't contain.
- If you find a gap in the CSP while drafting a PRD, stop and tell the user; extending the CSP is `clean-room-extract`'s job, not yours. Do not invent.
- PRDs inherit the CSP's contamination discipline: no source-language syntax, no leaked names from the original repo, no copied code or strings.

## Process

### 1. Inventory the CSP

Read every file in `csp/` in this order: `00-overview.md`, `01-architecture.md`, `02-public-api.md`, `03-behaviors.md`, then the rest. Build a mental map of: deep modules, public surface, behaviors, error/effect model, data shapes, test contracts, non-functionals.

If `csp/` is missing files, malformed, or clearly partial, stop and tell the user. The CSP must be in good shape before slicing into PRDs.

### 2. Frame the slicing

One short turn with the user. No relentless interview. Confirm:

- **Slicing strategy**: `hybrid` (default: foundation PRDs first, then vertical slices), `vertical` (pure end-to-end slices, no separate foundations), or `module` (one PRD per deep module from `01-architecture.md`).
- **Granularity**: roughly how many PRDs should the CSP yield? Suggest a number based on CSP size; let the user adjust. Default heuristic: 1 foundation + 1 PRD per deep module + 1 per major user-observable flow, capped around 8-12 unless the CSP is unusually large.
- **Output location**: `prds/` at the repo root (default), parallel to `csp/`.
- **Target paradigm carryover**: if `clean-room-extract` was run with a target paradigm (e.g. "functional + effect system"), reflect that in PRDs. Otherwise stay paradigm-neutral. Check `csp/00-overview.md` first; only ask the user if it's not recorded there.
- **Issue tracker hand-off**: files only (default), or also format for direct paste into GitHub/Linear/Jira issues? (Default: files. Issue submission is out of scope; emit clean markdown.)

If the user supplied this in their initial request, skip and proceed.

### 3. Plan the slices

Propose the PRD list as a numbered table: `id`, `title`, `one-line scope`, `depends on`. This is the only other place a brief check-in is warranted: confirm the slice list with the user in one turn before drafting.

A good slice has:

- A clear externally observable outcome (the build team can demo it)
- A small set of CSP behaviors it covers (cite them by section)
- Minimal coupling to other slices beyond declared dependencies
- Independent buildability: once its dependencies land, an engineer can grab it without needing context from sibling PRDs

If two proposed slices share more than 30-40% of their behaviors, they're probably one slice. If a single slice covers more than ~6 deep modules, split it.

Read `references/slicing-and-audit.md` for the full slicing strategy guide and the post-draft audit checklist.

### 4. Write the PRDs

Create the output directory (default `prds/`) and write one markdown file per slice using the template in `references/prd-template.md`. File naming: `00-foundations.md`, `01-<kebab-slice-name>.md`, `02-<kebab-slice-name>.md`, etc.

Also write `prds/00-INDEX.md` containing:

- One-paragraph build-team orientation (what's in this folder, how to use it)
- The full PRD list with dependencies rendered as a simple ASCII or mermaid DAG
- A note that the CSP (`csp/`) is the only source of truth; PRDs are scoping documents that point back into it

Each PRD MUST cite the CSP sections it derives from, e.g. "see `csp/03-behaviors.md` §Order Submission" or "implements `csp/02-public-api.md` §Payments module". Citations are how the build team finds the authoritative spec; without them, the PRDs become a parallel source that drifts.

### 5. Self-audit

Run the checklist in `references/slicing-and-audit.md`. Verify:

- **Coverage**: every section in `csp/02-public-api.md`, `03-behaviors.md`, and `07-test-contracts.md` is covered by at least one PRD. Anything in the CSP that no PRD picks up is either dead spec (remove from CSP) or missed scope (add a PRD).
- **No duplication**: no behavior is the responsibility of two PRDs. If a behavior is shared, factor it into the foundations PRD or an explicit shared-module PRD.
- **DAG, not graph**: the dependency relation is acyclic. Cycles mean the slicing is wrong; rewrite, don't patch.
- **No contamination**: no PRD contains source-language syntax, source identifier names beyond what's in the CSP, or facts not traceable to a CSP section.
- **No invention**: every claim in a PRD has a CSP citation or is clearly framed as build-team latitude (e.g. "implementation may choose any storage backend that satisfies the durability requirement").

If any check fails: rewrite the affected PRDs from scratch. Patching introduces drift between PRDs and CSP.

### 6. Hand off

Tell the user `prds/` is ready. State explicitly:

- The build team works from `prds/00-INDEX.md` first, then picks PRDs in dependency order.
- The CSP remains the source of truth for behavior; PRDs are scope and slicing.
- If the build team finds a CSP gap, the fix is to extend the CSP via `clean-room-extract`, then regenerate the affected PRDs, not to amend a PRD with new facts.

## Defaults for ambiguity

- When a CSP behavior could fit in two slices, put it in the slice with the smaller dependency footprint and cross-reference from the other.
- When the CSP describes a non-functional requirement (`08-non-functionals.md`) that affects multiple slices, capture it in the foundations PRD as a system-wide constraint and let downstream PRDs reference it.
- When the CSP has `[INTENT UNCLEAR]` flags from `clean-room-extract`, surface them in the relevant PRD's `Open Questions` section; never silently resolve them.
- When a slice's user stories are obvious from the CSP behaviors, write them anyway. The PRD is the build team's full context for the slice; don't make them cross-reference unnecessarily.

## What this skill is not

- Not a clean-room **extract**: that's `clean-room-extract`. If the CSP doesn't exist or is incomplete, run that first.
- Not a clean-room **build**: you do not implement anything. PRDs are scope documents.
- Not an issue tracker integration: the output is markdown files. Pasting them into GitHub/Linear/Jira is the user's call.
- Not a project plan: PRDs declare dependencies but not estimates, assignees, or sprint allocations.

## See also

- `references/prd-template.md`: the canonical PRD structure (read at step 4)
- `references/slicing-and-audit.md`: slicing strategies and the audit checklist (read at step 3, re-read at step 5)
