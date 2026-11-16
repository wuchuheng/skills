---
name: code-complete-typescript
description: Review and improve TypeScript readability and maintainability using Code Complete principles, with attention to conditional complexity, naming, routine cohesion, reuse, domain literals, type boundaries, comments, exception handling, and safe refactoring. Use for substantive readability reviews or cleanup; not formatting-only requests or a general security audit.
---

# Code Complete for TypeScript

Make the code's intent, permitted states, effects, and failure paths understandable without reconstructing them mentally. This is a practical adaptation of Steve McConnell's *Code Complete*, second edition, not an official compliance standard or a replacement for project conventions.

## Choose the requested work

- **Review:** Inspect the requested code, callers, contracts, and tests. Report findings without editing.
- **Refactor:** Make authorized, bounded readability improvements while retaining observable behavior.
- **Teach:** Explain the difficulty and show an original TypeScript before/after example, including its assumptions and tradeoffs.

Do not expand a focused request into a repository rewrite. Do not interpret a request to assess code as authorization to change its API or business behavior.

Read [criteria.md](references/criteria.md) for the review criteria. Consult only relevant sections of [typescript-examples.md](references/typescript-examples.md) when choosing a repair. Read [research.md](references/research.md) when citing the book or distinguishing book guidance from modern TypeScript adaptations.

## Required working discipline

1. Establish the observable contract before editing: inputs, outputs, exceptions, mutations, side-effect order, async ordering, cancellation, and resource ownership. Inspect callers when available. For a pasted snippet without caller access, give a bounded review and state assumptions and missing evidence; a suspicious pattern is not automatically a defect.
2. Name the concrete comprehension problem and choose the smallest repair that removes it. Prefer a meaningful name, guard, or domain type over a new abstraction layer. A helper earns its place by expressing a concept or isolating a change, not merely by shortening its caller.
3. Separate **refactoring**, **bug fixes**, and **API/model changes**. Identify the latter two explicitly and keep them within the user's authorization. Do not silently change defaults, validation strictness, callback timing, error handling, or parallelism during cleanup.
4. Preserve short-circuit evaluation and side-effect counts. Do not eagerly calculate named booleans if the original expression skipped those operations. Preserve evaluation order for getters, effectful functions, and mutable state.
5. Keep `unknown` for genuinely untrusted data; narrow or parse it before domain use. Replace unnecessary internal `unknown`, `any`, and casts with meaningful types. Never replace runtime validation with an assertion just to obtain a cleaner signature.
6. Keep refactorings small and reviewable. Preserve existing work and assertions. Run the checks proportionate to the changed boundary: focused behavior tests, typecheck, lint/format, and integration/build checks when adapters or packaging are affected. Add a regression case for an uncovered defect, not tests that merely mirror renamed helpers.
7. For a refactor, stop when the requested scope is clearer and verified; report what improved, preserved behavior, checks actually run, and unresolved limitations. For review-only work, report recommendations and evidence limits without implying edits or requiring execution unavailable to the review. No automatic commits or unrelated configuration changes.

## Required encapsulation and reuse review

Before polishing a substantial logic block, consider whether it represents a cohesive operation that should become a named function. Reusing the same parameters does not make extraction pointless; shared inputs can reveal a missing domain operation. Look for existing canonical helpers before creating another implementation. Repeated logic expressing the same rule should normally have one owner; a clear single-use operation can also justify extraction. Do not wait for an arbitrary third occurrence.

Inspect repeated literals for shared meaning and ownership. Consider a named constant, literal union, `as const` value object, or enum rather than repeating domain values. Equal spelling or numeric value alone is not enough to merge independent policies. Apply [encapsulation-and-reuse.md](references/encapsulation-and-reuse.md) to select the repair and preserve effects, dependencies, and wire values. The review must consider extraction and reuse; it must not mechanically extract every block or convert every literal to an enum.

## Required exception-handling review

When scoped code can throw, reject, recover, or own resources, apply [exception-handling.md](references/exception-handling.md). Check the failure contract, recovery/propagation owner, abstraction and diagnostic safety, cleanup outcomes, and asynchronous rejection ownership. Treat silent failure, accidental success, masked primary failures, and unowned rejections as correctness findings when reachable. Do not require try/catch in every function, impose a new error hierarchy, or change failure behavior merely to satisfy a style preference.

## Required commenting review

For scoped JS/TS/TSX code, apply [commenting.md](references/commenting.md). Document public contracts and non-obvious constraints, failure behavior, ownership, and ordering. Prefer clearer names or structure over comments that narrate mechanics. Verify comments against implementation and tests; remove stale claims. Preserve the user-required step prefixes: `1. Input`, `2. Processing`, and `3. Return`. Use numbered phase comments for multi-step routines and substeps for detail; do not remove them as redundant style. See the reference for early returns, asynchronous work, and cleanup. Comment-only work must preserve executable behavior and compiler/linter directives.

## Review output

For each material finding give: severity, location, the reader's specific difficulty, its maintenance consequence, the proposed repair, and the behavior that must remain unchanged. Distinguish a defect from an optional style preference. When useful, show a small TypeScript before/after pair; do not rewrite an entire file merely to illustrate one finding.

Review completion requires evidence, not a numeric readability score. Long conditions, large functions, many parameters, or deep nesting are inspection signals—not automatic failures. Do not invent universal line counts, maximum name lengths, mandatory comment density, a ban on `unknown`, or a requirement that every function be generic.
