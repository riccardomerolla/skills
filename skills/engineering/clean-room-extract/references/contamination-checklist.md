# Contamination Checklist

Run this against every file in `csp/` before declaring the extract complete. If any check fails: do **not** patch. Rewrite the offending file from scratch using only your mental model of the system's behavior.

## Verbatim code

- [ ] No source code is quoted, in any form — no functions, classes, expressions, not even one-liners.
- [ ] Type signatures are written in pseudo-syntax (records, unions, function arrows). Not in the source language's literal syntax with all its keywords intact. `submit(order: Order) -> Result<Receipt, SubmitError>` is fine; `def submit(order: Order): IO[SubmitError, Receipt]` is contamination.
- [ ] Configuration files (`package.json`, `tsconfig.json`, `application.conf`, etc.) are not embedded. Their *intent* is described instead ("the service reads three settings: …").
- [ ] No SQL/GraphQL/protobuf schemas are quoted verbatim from migration files or `.proto` definitions. Re-express the schema in plain field/type notation.

## Distinctive expression

- [ ] No comments from the source are quoted or paraphrased closely. Paraphrase the *meaning*, not the wording.
- [ ] No log message, error message, or string literal is reproduced verbatim. Describe the *trigger condition*, not the literal string.
- [ ] No README/docs prose is paraphrased so closely that a reader could reconstruct the original. Reformulate from scratch using the underlying facts.
- [ ] No commit messages or PR descriptions appear in the CSP.

## Naming

- [ ] Distinctive identifier names — project-branded, trademarked, or carrying expressive content — are renamed. The mapping is recorded in `04-domain-glossary.md`.
- [ ] Generic names (`User`, `Order`, `Repository`, `Service`) may be kept if they're industry-standard and the source didn't pick them as a distinctive choice.
- [ ] Internal helper names do not appear at all — they're not part of the public contract, and including them creates avoidable similarity.
- [ ] File paths from the source repo do not appear in the CSP. The CSP is organized around *modules*, not files.

## Algorithmic structure

- [ ] No section reads like "first do X, then Y, then Z" tracing the original source's order of operations. Specify *required outcomes*, not implementation steps.
- [ ] Where an algorithm is genuinely part of the contract (a documented hashing scheme, a published wire protocol, a standardized format), describe its *external behavior* and reference the standard — don't transcribe the source's implementation.
- [ ] Loop structures, recursion patterns, and control-flow shapes from the source are not preserved. The build team designs control flow.

## Public surface only

- [ ] Only the public API is documented. Internal modules, private helpers, build orchestration, and the source's directory structure are absent.
- [ ] Tests in `07-test-contracts.md` are described in black-box terms (input → expected outcome). Source test names, assertion messages, and test framework idioms (`describe`/`it`, `should`, `@Test`) do not appear.
- [ ] No mention of the source's specific dependencies unless those dependencies define an externally observable behavior the build team must preserve (e.g. "requests follow OAuth 2.0" — yes; "uses `passport-oauth2` v3.4.1" — no).

## Secrets and operational data

- [ ] No keys, tokens, credentials, internal URLs, IP ranges, or other secrets from `.env` or config files appear anywhere in the CSP.
- [ ] No customer names, account IDs, or PII from fixtures or seed data appear, even paraphrased.

## Final smell test

Read each CSP file as if you were the build team. Ask:

1. Could a competent engineer implement this section *without* needing access to the source repo?
2. If they did implement it independently, would their result be functionally equivalent?
3. Is there anything in this file that I only know because I read the source — and that the build team would have no way of knowing without also reading the source?

If the answer to (3) is yes, that's contamination by *implicit* reference even if no text was copied. Restate the behavior in terms the build team could verify externally.

## Failure mode

If any check fails: rewrite. Don't patch. The whole value of the CSP is that it has a clean provenance — a patched file with traceable origins defeats the purpose.
