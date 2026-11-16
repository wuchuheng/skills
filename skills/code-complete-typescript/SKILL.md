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
6. Keep refactorings small and reviewable. Preserve existing work and assertions. After authorized fixes, complete the required format → lint → tests loop below. Focused tests can help diagnose failures but do not replace the supported project test command. Also run required typechecks and integration/build checks when the changed boundary calls for them. Add a regression case for an uncovered defect, not tests that merely mirror renamed helpers.
7. For a refactor, stop when the requested scope is clearer and verified; report what improved, preserved behavior, checks actually run, and unresolved limitations. For review-only work, report recommendations and evidence limits without implying edits or requiring execution unavailable to the review. No automatic commits or unrelated configuration changes.

## Required encapsulation and reuse review

Before polishing a substantial logic block, consider whether it represents a cohesive operation that should become a named function. Reusing the same parameters does not make extraction pointless; shared inputs can reveal a missing domain operation. Look for existing canonical helpers before creating another implementation. Repeated logic expressing the same rule should normally have one owner; a clear single-use operation can also justify extraction. Do not wait for an arbitrary third occurrence.

Inspect repeated literals for shared meaning and ownership. Consider a named constant, literal union, `as const` value object, or enum rather than repeating domain values. Equal spelling or numeric value alone is not enough to merge independent policies. Apply [encapsulation-and-reuse.md](references/encapsulation-and-reuse.md) to select the repair and preserve effects, dependencies, and wire values. The review must consider extraction and reuse; it must not mechanically extract every block or convert every literal to an enum.

## Required exception-handling review

When scoped code can throw, reject, recover, or own resources, apply [exception-handling.md](references/exception-handling.md). Check the failure contract, recovery/propagation owner, abstraction and diagnostic safety, cleanup outcomes, and asynchronous rejection ownership. Treat silent failure, accidental success, masked primary failures, and unowned rejections as correctness findings when reachable. Do not require try/catch in every function, impose a new error hierarchy, or change failure behavior merely to satisfy a style preference.

## Required commenting review

For scoped JS/TS/TSX code, apply [commenting.md](references/commenting.md). Document public contracts and non-obvious constraints, failure behavior, ownership, and ordering. Prefer clearer names or structure over comments that narrate mechanics. Verify comments against implementation and tests; remove stale claims. Preserve the user-required step prefixes: `1.`, `2.`, and `3.`. Use numbered phase comments for multi-step routines and substeps for detail; do not remove them as redundant style. See the reference for early returns, asynchronous work, and cleanup. Apply the required [logical-block spacing rules](references/logical-block-spacing.md): separate major phases and distinct substeps with a blank line, while keeping each cohesive group together. Comment and spacing edits must preserve executable behavior and compiler/linter directives.

## Required verification and repair loop

After making authorized fixes, inspect the relevant project's `package.json`, package-manager declaration, lockfile, and repository guidance. Use its existing format, lint, and test scripts with the established package manager and working directory. Check documented equivalents when a script has another name. In a workspace, use its prescribed root orchestration or affected-package commands. Do not invent scripts or install a new toolchain solely to satisfy this loop.

1. **Format:** Execute the project's format command if supported. Inspect any changes it makes. If it fails, diagnose and fix the cause, then restart at step 1. A formatting check does not replace the project's formatting command when a formatter is available.
2. **Lint:** Once formatting succeeds (or is unsupported), execute the project's lint command if supported. Fix lint failures, then return to step 1. Do not proceed to tests while a supported formatting or lint step still fails. Lint autofixes that modify files also require a restart from step 1.
3. **Tests:** Once the earlier supported steps pass, execute the project's test command if supported, using its documented non-watch/CI mode when necessary. If tests fail, diagnose and fix the cause, then return to step 1 so formatting and lint are rechecked before testing again.
4. **Finish:** Stop only after the supported steps pass in order against the final files. If a later fix, typecheck, build, or integration check changes those files, restart at step 1. Report the actual commands and results, plus unsupported or blocked steps. Unsupported commands are skipped explicitly, never reported as passed.

Continue this repair loop until the failures are resolved; do not stop after merely reporting a fixable lint or test error. Diagnose between attempts rather than rerunning unchanged commands indefinitely. Preserve assertions and meaningful checks: do not skip failing tests, weaken expectations, disable lint rules, or alter scripts just to obtain a successful exit code.

Respect the authorized scope and existing user changes. Investigate failures outside the edited code rather than assuming they predate the work. If resolving a failure requires an unrelated behavior change or unavailable external resources, identify the evidence and concrete blocker, state that verification is incomplete, and request only the missing information or authorization. A missing service or credential is not a passed test and cannot be repaired by retrying forever.

This loop applies to implementation and repair work. A review-only request remains read-only: do not run a mutating formatter or make fixes without authorization. A teaching-only snippet need not acquire a new project scaffold. These workflow requirements are user-defined verification policy, not claims about commands prescribed by Code Complete.

For example, in a pnpm project exposing `format`, `lint`, and a one-shot `test` script, a test repair requires the full sequence again:

```text
pnpm run format → pnpm run lint → pnpm run test
                                      ↓ failure
                                  diagnose and fix
                                      ↓
pnpm run format → pnpm run lint → pnpm run test → all pass
```

## Review output

For each material finding give: severity, location, the reader's specific difficulty, its maintenance consequence, the proposed repair, and the behavior that must remain unchanged. Distinguish a defect from an optional style preference. When useful, show a small TypeScript before/after pair; do not rewrite an entire file merely to illustrate one finding.

Review completion requires evidence, not a numeric readability score. Long conditions, large functions, many parameters, or deep nesting are inspection signals—not automatic failures. Do not invent universal line counts, maximum name lengths, mandatory comment density, a ban on `unknown`, or a requirement that every function be generic.
