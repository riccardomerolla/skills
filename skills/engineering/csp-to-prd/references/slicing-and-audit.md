# Slicing Strategies & Audit Checklist

Read at step 3 (planning slices). Re-read at step 5 (self-audit).

## Slicing strategies

### Hybrid (default)

One **foundations** PRD first, then **vertical slices** on top. Best for most CSPs; gives the build team something concrete to grab in week one without forcing them to design the whole system first.

The foundations PRD covers:

- Cross-cutting data shapes from `csp/06-data-model.md` that multiple slices need (shared entities, identifier types, common envelopes)
- The system-wide error model from `csp/05-effects-and-errors.md` (typed error categories, retry semantics, propagation rules)
- Cross-cutting non-functional requirements from `csp/08-non-functionals.md` (auth, observability hooks, idempotency conventions)
- The build conventions the rest of the slices will conform to (module boundaries, public-API style, test layout)

Each subsequent PRD is a vertical slice — a thin path through the system that delivers an externally observable behavior end-to-end. Order them by dependency: a slice goes earlier if it unblocks more downstream slices.

Don't put implementation logic into foundations. Foundations are *shared scaffolding*. If a piece of behavior is only used by one slice, it belongs to that slice, not foundations.

### Vertical (pure end-to-end)

No foundations PRD. Every PRD is a vertical slice. The first slice carries the cross-cutting concerns alone, and later slices extend or reuse what it established.

Use this when:

- The CSP is small (≤ 4-5 modules) and a separate foundations PRD would feel ceremonial
- The team is small enough that the same engineer will pick up consecutive PRDs and doesn't need a separate handoff for shared code
- The CSP has minimal cross-cutting state — most behaviors live in their own modules

The risk: the first vertical slice gets bloated. Watch for that during planning.

### Module (one PRD per deep module)

One PRD per deep module from `csp/01-architecture.md`. Tracks the architecture exactly. Useful when the CSP describes a library or framework rather than an end-user product, where "vertical slice" doesn't map cleanly because there are no end-to-end user flows.

Use this when:

- The CSP is for a library, SDK, or platform component
- The architecture is genuinely modular and modules can be built and tested in isolation
- The build team is structured around module ownership rather than feature delivery

The risk: integration is deferred. Add an explicit final integration PRD that exercises the modules together.

## Picking the right strategy

Default to **hybrid**. Switch to **vertical** if the CSP is small or the foundations PRD would only contain 2-3 items. Switch to **module** only when there's no end-user product to deliver vertically.

Don't mix strategies within one PRD set. Pick one and apply it consistently — the build team reads the dependency graph as a whole, and mixed strategies make it incoherent.

## Granularity heuristics

For a CSP with `M` deep modules and `B` major user-observable behaviors:

- **Hybrid**: roughly `1 + M` PRDs at minimum, expanding by behavior count when modules host multiple distinct flows. Cap at ~12 unless the CSP is unusually large.
- **Vertical**: roughly `B` PRDs, one per major flow. Cap at ~8 unless the CSP describes many disjoint flows.
- **Module**: exactly `M` PRDs plus one integration PRD.

A PRD smaller than ~150 words of real content is too small — fold it into a sibling. A PRD that spans more than ~6 deep modules is too large — split by sub-flow or by cross-cutting vs. feature-specific.

## Dependency graph patterns

Healthy:

- A linear chain (`00 → 01 → 02 → 03`) — fine for small CSPs but slows parallel work
- A fan-out from foundations (`00 → {01, 02, 03}` with later PRDs joining branches) — best for parallel build teams
- A DAG with multiple roots — only valid when the CSP genuinely has independent subsystems with no shared scaffolding

Unhealthy:

- Cycles (any A → B → A) — slicing is wrong; rewrite
- Single-dependency chains where each PRD blocks exactly one other — usually a sign that what looks like 5 slices is really 1 slice in 5 pieces
- A dependency on "all earlier PRDs" — means the slice is a system-wide change and isn't actually a slice

## Audit checklist

Run every check before declaring `prds/` complete. Failures require **rewrite, not patch** — the same discipline as the parent CSP.

### Coverage

- [ ] Every module in `csp/01-architecture.md` is covered by at least one PRD's Implementation Decisions.
- [ ] Every public-API entry in `csp/02-public-api.md` is covered by at least one PRD.
- [ ] Every behavior in `csp/03-behaviors.md` appears in at least one PRD's User Stories or Implementation Decisions.
- [ ] Every test scenario in `csp/07-test-contracts.md` is referenced by at least one PRD's Testing Decisions.
- [ ] Every CSP `[INTENT UNCLEAR]` flag is surfaced as an Open Question in at least one PRD.

If any item in the CSP isn't covered: either add a PRD to cover it, or — if the user agrees — mark the item as dead spec and remove it from the CSP. Don't leave orphans.

### Disjointness

- [ ] No behavior from `csp/03-behaviors.md` is the primary responsibility of more than one PRD. If a behavior is shared infrastructure, it belongs in foundations (or an explicit shared PRD); other PRDs reference it but don't re-own it.
- [ ] No PRD's User Stories duplicate another PRD's User Stories.
- [ ] Out-of-Scope sections don't contradict each other (PRD A saying "X is out, see PRD B" while PRD B doesn't actually cover X).

### Dependency graph

- [ ] The `Depends on` fields form a DAG. No cycles. Verify by topological sort if needed.
- [ ] Every PRD listed in any `Depends on` field actually exists.
- [ ] No PRD depends on every earlier PRD. That's a system-wide change masquerading as a slice.
- [ ] `prds/00-INDEX.md` renders the graph correctly and matches the per-PRD `Depends on` fields.

### Contamination

- [ ] No PRD contains source-language syntax. Pseudo-syntax only, matching the CSP's notation.
- [ ] No PRD references source repo file paths. Only CSP citations.
- [ ] No PRD names a specific framework, library, or runtime primitive — even when describing required capabilities. ("requires a logging capability" yes; "uses `slf4j`" no.)
- [ ] No PRD quotes verbatim text from the source — only from the CSP, and even then only via citation, not copy-paste.

### No invention

- [ ] Every claim in a PRD is traceable to a CSP section via citation, OR is explicitly framed as build-team latitude (e.g. "implementation may choose any approach satisfying X").
- [ ] No PRD introduces a behavior the CSP doesn't establish.
- [ ] No PRD introduces a data shape the CSP doesn't establish.
- [ ] No PRD prescribes an algorithm. Contracts and required outcomes only.

### Self-containment

- [ ] Each PRD is readable by an engineer who has read only its dependency PRDs and the CSP. They should not need to read sibling PRDs.
- [ ] Each PRD's Out of Scope is specific enough that the engineer can confidently set boundaries without sideways coordination.
- [ ] Each PRD's Open Questions are framed as decisions the engineer can make (or escalate) — not as research tasks.

### Smell test

Read `prds/00-INDEX.md` and the first PRD as if you were the build team's lead engineer. Ask:

1. Could I assign these PRDs to engineers and expect them to proceed in parallel after foundations land?
2. If two engineers were working on adjacent PRDs at the same time, would they collide on shared assumptions?
3. Is there anything in any PRD that I only know because I read the source — and that the build team would have no way to verify without also reading the source?

If the answer to (3) is yes, you've leaked across the clean-room boundary. Rewrite the offending PRD.

## When the CSP itself is wrong

If the audit reveals a real CSP gap (a behavior that needs to be implemented but isn't specified, or two CSP sections that contradict each other), stop and tell the user. Do not paper over it in the PRDs. The CSP is the contract; PRDs derive from it. A PRD that compensates for a broken CSP creates two sources of truth, and the build team will end up implementing the wrong thing.

The fix is to extend or correct the CSP via `clean-room-extract`, then regenerate the affected PRDs.
