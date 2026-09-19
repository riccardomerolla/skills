---
name: clean-room-extract
description: Reverse-engineer a git repository into a Clean Specification Pack (CSP): a language-agnostic, license-clean set of behavioral specifications a separate build team can use to reimplement the system from scratch without ever seeing the source. Use whenever the user wants to extract specs from a codebase, perform the "extract" phase of a clean-room operation, document a system's contract for a port or reimplementation, escape AGPL/copyleft contamination, build a legally distinct reimplementation, or says things like "clean-room this repo", "extract the spec", "reverse engineer to spec", "to-csp", or mentions clean-room operations even casually.
---

# Clean-Room Extract

Take a git repo as input. Produce a **Clean Specification Pack (CSP)** as output: a language-agnostic, paradigm-neutral, license-clean description of the system's observable behavior, sufficient for a build team that has never seen the source to reimplement it from scratch.

You are the **extract team**. A separate, isolated **build team**, who will never read the original source, uses your CSP as their only context. The CSP must therefore describe *what* the system does and *why*, never *how* it is currently coded.

## The contamination rule

The CSP MUST contain:

- ✅ Behaviors, contracts, invariants, types-as-shapes, observable effects
- ✅ Generic domain terms; renamed equivalents for distinctive or branded ones
- ✅ Black-box test scenarios derived from observed behavior
- ✅ Architectural intent ("this layer mediates between X and Y")

The CSP MUST NOT contain:

- ❌ Verbatim code: no copy-paste, not even a single line "for reference"
- ❌ Implementation algorithms transcribed in source-language form
- ❌ Comment text, log strings, or error messages copied as written; paraphrase
- ❌ Distinctive identifier names that carry expressive content; neutralize them
- ❌ Internal helpers and private code; only the public surface is the contract

If you find yourself wanting to quote, transcribe, or preserve a specific phrasing: stop. That's contamination. Restate it.

Read `references/contamination-checklist.md` before you start writing the CSP, and again at the audit step.

## Process

### 1. Inventory

Walk the repo. Read in this order: `README`, top-level docs, the manifest (`package.json` / `build.sbt` / `Cargo.toml` / `pyproject.toml` / etc.), the public module/package surface, types and interfaces, then tests. Skim, don't transcribe.

Note: language(s), ecosystem, primary dependencies, rough size, license(s), test framework. Do **not** read internal/private modules unless explicitly necessary to infer public behavior you couldn't otherwise see.

If the docs are sparse, switch to behavior reconstruction from tests + types; see `references/extraction-playbook.md`.

### 2. Frame the scope

One short turn with the user. No relentless interview. Confirm:

- **Scope**: whole repo, a specific module, or just the public API?
- **Target paradigm**: paradigm-agnostic CSP (default), or explicitly written to fit a target like "functional + effect system", "actor model", or "event-driven services"?
- **Renaming policy**: keep generics, rename distinctive terms (default), or rename everything?
- **Tests**: derive black-box test contracts? (Default: yes.)
- **Output location**: `csp/` at the repo root (default), or somewhere else?

If the user already supplied this in their initial request, skip and proceed.

### 3. Map deep modules

Identify **deep modules**: pieces that hide a lot of complexity behind a small interface. Organize the CSP around these, not around source files. A deep module is a candidate when its public interface fits on a page and its behavior takes more.

Confirm the module list with the user in one turn before drafting the CSP. This is the only other place a brief check-in is warranted; everything else proceeds without further interview.

### 4. Write the CSP

Create the output directory (default `csp/`) and write one markdown file per section:

```
csp/
├── 00-overview.md           # system purpose and high-level behavior (one page)
├── 01-architecture.md       # deep modules, boundaries, dataflow
├── 02-public-api.md         # signatures and shapes; pseudo-syntax, not source
├── 03-behaviors.md          # input → output rules, state transitions, invariants
├── 04-domain-glossary.md    # terms used; renames where applicable
├── 05-effects-and-errors.md # I/O, concurrency, failure modes
├── 06-data-model.md         # entities, relationships, schemas
├── 07-test-contracts.md     # black-box scenarios, no source tests copied
├── 08-non-functionals.md    # performance, security, ops constraints
└── 09-out-of-scope.md       # what the build team MUST NOT recreate
```

Skip a section only if it genuinely doesn't apply. Note the skip in `00-overview.md`.

Express types as shapes: records of fields, tagged unions, function signatures rendered in plain notation like `submit(order: Order, idempotencyKey: string) -> Result<Receipt, SubmitError>`. Do not use any one source language's literal syntax (no Scala `case class`, no TypeScript `interface`, no Python `@dataclass`). The build team picks the language.

### 5. Self-audit for contamination

Run `references/contamination-checklist.md` against every file. If any check fails, **rewrite, don't patch**. Patching tends to leave fingerprints; rewriting is clean.

### 6. Hand off

Tell the user the CSP is ready. State explicitly: the build team should now work *only* from `csp/`, with no access to the source repo. The next clean-room phases (build, rewrite) are out of scope for this skill.

## Defaults for ambiguity

- When the source uses a generic name (`User`, `Order`, `Repository`) keep it. When it uses a distinctive one (project-branded, trademarked, vendored library names) rename it and record the mapping in `04-domain-glossary.md`.
- When you can't tell whether a behavior is intentional or incidental, capture it as observed and flag it for the build team's discretion in `09-out-of-scope.md` or as a note in the relevant section.
- When the source is sparse, prefer fewer well-stated behaviors over many speculative ones. The build team shouldn't have to guess what you meant.
- When the source contains a clever optimization, document the *required outcome* (e.g. "lookup must be O(log n) or better") and let the build team choose how to achieve it.

## Safety

- Do not execute code from the repo. Reading is enough to extract behavior.
- Do not include the user's API keys, tokens, internal URLs, or other secrets discovered in `.env` or config files anywhere in the CSP.
- This skill reduces contamination risk; it is **not legal advice**. A real clean-room programme also requires documented team isolation, audit logs of who saw what, and review by qualified counsel. Tell the user this when handing off.

## What this skill is not

- Not a clean-room **build**: you do not implement the replacement.
- Not a license audit: point users at SPDX / `license-checker` / `licensee` for that.
- Not a security review: vulnerabilities in the source are not the CSP's concern.
- Not a moralizer: the technique is legitimate (it's how IBM PC BIOS got cloned). Don't lecture the user about open-source ethics; just do the work well.

## See also

- `references/contamination-checklist.md`: the audit checklist (re-read at step 5)
- `references/extraction-playbook.md`: techniques for behavior reconstruction when docs are sparse
