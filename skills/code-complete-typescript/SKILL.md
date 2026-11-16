---
name: code-complete-typescript
description: Review and improve TypeScript readability and maintainability using Code Complete principles, with attention to conditional complexity, naming, routine cohesion, reuse, domain literals, type boundaries, comments, exception handling, and safe refactoring. Default to reviewing and directly repairing applicable issues unless the user requests read-only review or teaching. Use for substantive readability reviews or cleanup; not formatting-only requests or a general security audit.
---

# Code Complete for TypeScript

Make the code's intent, permitted states, effects, and failure paths understandable without reconstructing them mentally. This is a practical adaptation of Steve McConnell's *Code Complete*, second edition, not an official compliance standard or a replacement for project conventions.

## Choose the requested work

- **Review and repair (default):** Inspect the target, create the required checklist, and directly fix applicable failures. Invoking this skill without an explicit mode, including a general request to check the current changes, does not require a separate approval question before ordinary scoped repairs. Continue until all applicable checklist items pass and supported verification commands succeed.
- **Review only:** When the user explicitly asks for findings only, no edits, read-only analysis, or a review artifact without modifications, inspect and report without changing files. Explicit user constraints take precedence over the default.
- **Teach:** When the request is explanation or examples only, explain the difficulty and show original TypeScript before/after examples without editing the project.

Do not expand a focused request into a repository rewrite. Default repair covers scoped readability, maintainability, and evidence-backed correctness fixes that preserve established contracts. Identify bug fixes separately. It does not authorize inventing business requirements, redesigning public APIs, changing intended behavior to satisfy tests, committing, pushing, or publishing. Ask only when a material ambiguity or action outside that scope prevents progress; do not ask again for permission to perform ordinary checklist repairs.

Read [criteria.md](references/criteria.md) for the review criteria. Consult only relevant sections of [typescript-examples.md](references/typescript-examples.md) when choosing a repair. Read [research.md](references/research.md) when citing the book or distinguishing book guidance from modern TypeScript adaptations.

## Default scope and mandatory checklist

If the user specifies files, a directory, a diff, or a revision range, use that scope. Otherwise, default to the current repository's changed files: staged changes, unstaged changes, and relevant untracked files. Inspect `git status --short`, `git diff --cached`, `git diff`, and untracked file contents; do not assume `git diff` alone includes all changes. Use rename/deletion information and compare against HEAD where available (an unborn repository has no HEAD). Do not stage, reset, or discard files merely to review them.

Review changed hunks in their enclosing routines and types. Read callers, contracts, and tests as needed to establish behavior; this contextual reading does not authorize unrelated edits. Exclude generated/vendor files from manual rewriting and report any relevant excluded files. Honor explicit narrower scope, including staged-only requests. If the worktree is clean, report that there are no current changes and ask for a target only if needed; do not silently review the whole repository or choose a commit range. Outside Git, use provided files or ask for the missing scope.

Before reviewing or fixing, instantiate the [mandatory requirement checklist](references/review-checklist.md) as a user-visible table with one row per requirement ID. A private tracker or list of categories is not a substitute. Publish the initial assessment before repairs, update failed/pending items as work proceeds, and publish the complete final table with initial status, final status, and evidence for each ID. Evaluate every requirement against the target; record applicability and evidence, not just a blanket promise to follow the skill. Do not create a repository tracking file unless requested or required by the project.

In explicit review-only mode, record findings without edits. Otherwise, turn applicable failures into repair tasks immediately, complete them, and recheck the full applicable checklist against the final changes without waiting for separate fix approval. Reinspect status/diffs to include newly edited files. Passing format/lint/tests does not replace this manual requirement check. Do not declare all issues fixed while applicable failures, pending checks, or blockers remain.

## Required working discipline

1. Establish the observable contract before editing: inputs, outputs, exceptions, mutations, side-effect order, async ordering, cancellation, and resource ownership. Inspect callers when available. For a pasted snippet without caller access, give a bounded review and state assumptions and missing evidence; a suspicious pattern is not automatically a defect.
2. Name the concrete comprehension problem and choose the smallest repair that removes it. Prefer a meaningful name, guard, or domain type over a new abstraction layer. A helper earns its place by expressing a concept or isolating a change, not merely by shortening its caller.
3. Separate **refactoring**, **bug fixes**, and **API/model changes**. Identify the latter two explicitly and keep them within the user's authorization. Do not silently change defaults, validation strictness, callback timing, error handling, or parallelism during cleanup.
4. Preserve short-circuit evaluation and side-effect counts. Do not eagerly calculate named booleans if the original expression skipped those operations. Preserve evaluation order for getters, effectful functions, and mutable state.
5. Keep `unknown` for genuinely untrusted data; narrow or parse it before domain use. Replace unnecessary internal `unknown`, `any`, and casts with meaningful types. Never replace runtime validation with an assertion just to obtain a cleaner signature.
6. Keep refactorings small and reviewable. Preserve existing work and assertions. After fixes, complete the required format → lint → tests loop below. Focused tests can help diagnose failures but do not replace the supported project test command. Also run required typechecks and integration/build checks when the changed boundary calls for them. Add a regression case for an uncovered defect, not tests that merely mirror renamed helpers.
7. For a refactor, stop when the requested scope is clearer and verified; report what improved, preserved behavior, checks actually run, and unresolved limitations. For review-only work, report recommendations and evidence limits without implying edits or requiring execution unavailable to the review. No automatic commits or unrelated configuration changes.

## Mandatory names and nesting limit

Every variable and constant name must communicate its domain meaning and role at its point of use. Review parameters, destructured bindings, loop variables, callbacks, caught errors, and static/module values; rename ambiguous bindings rather than relying on comments to decode them. Follow [names-and-nesting.md](references/names-and-nesting.md) and checklist R2.

Control-flow nesting must be **three levels or fewer within each function or method**, including callbacks. A fourth level fails checklist R6 and must be refactored in default repair mode. Use safe guard clauses or cohesive named helpers with explicit inputs; preserve order, failure behavior, and cleanup. Count levels using the reference, not raw braces or line indentation. This is an explicit user requirement, not a universal threshold attributed to Code Complete.

## Mandatory import paths and ordering

Apply [import-paths.md](references/import-paths.md) and checklist I1–I2. Keep `@/` mapped exclusively to `src/`; do not add other aliases or test-directory overrides for this rule. Retain relative paths for targets outside `src/`. Before rewriting or retaining an import targeting `src/`, derive both relative and `@/` candidates, verify that they resolve to the same permitted module, compare their lengths, and record the decision. Choose the shorter readable valid form. Keep an existing relative path when the alias would be longer or equal; alias usage alone is not a goal. If the missing `@/ → src/` mapping would shorten scoped imports, verify the directory and affected resolvers, then configure it directly in default repair mode. Preserve conflicting existing mappings and report the incompatibility; do not repoint them. Do not add alias configuration when it provides no shorter target paths. Explicit review-only restrictions still apply. Do not claim support based on TypeScript paths alone. Sort imports using the project's established convention while preserving side-effect order, runtime resolution, and public feature boundaries.

## Required encapsulation and reuse review

Before polishing a substantial logic block, consider whether it represents a cohesive operation that should become a named function. Reusing the same parameters does not make extraction pointless; shared inputs can reveal a missing domain operation. Look for existing canonical helpers before creating another implementation. Repeated logic expressing the same rule must have one canonical implementation; refactor its duplicate consumers to reuse it rather than merely suggesting reuse. A clear single-use operation can also justify extraction. Do not wait for an arbitrary third occurrence.

Scan the target and owning feature for repeated logic and literals, including switch cases, comparisons, payload constructors, defaults, runtime allowlists, and tests. Once two occurrences represent the same domain value or rule, consolidation is mandatory: define or reuse a canonical constant, `as const` value object, enum, or cohesive helper and migrate the relevant consumers. A type-only literal union does not remove duplicated runtime literals. Do not mark the requirement passed merely because reuse was considered.

Apply [encapsulation-and-reuse.md](references/encapsulation-and-reuse.md) and record a duplication inventory with locations, meaning, owner, chosen repair, and verification. Preserve wire values, public contracts, effect order, and failure semantics. Place the definition in the narrowest common owning layer; consumers must not import an implementation layer solely to get a constant. Equal values with independent meanings require separate semantic owners, not artificial coupling. Any retained duplicate must have a concrete, evidence-backed reason such as independent protocol-test expectations or an incompatible ownership boundary; “only twice,” “small,” or “tests pass” are not exemptions.

## Required exception-handling review

When scoped code can throw, reject, recover, or own resources, apply [exception-handling.md](references/exception-handling.md). Require checklist E3 for every catch and rejection handler: recover locally when correct, otherwise propagate, and report once at the owning boundary. Silent suppression needs an implementation-backed explanation of why it is safe; a comment alone is not sufficient. Check the failure contract, recovery/propagation owner, abstraction and diagnostic safety, cleanup outcomes, and asynchronous rejection ownership. Treat silent failure, accidental success, masked primary failures, and unowned rejections as correctness findings when reachable. Do not require try/catch in every function, impose a new error hierarchy, or change failure behavior merely to satisfy a style preference.

## Required commenting review

For scoped JS/TS/TSX code, apply [commenting.md](references/commenting.md). Document public contracts and non-obvious constraints, failure behavior, ownership, and ordering. Prefer clearer names or structure over comments that narrate mechanics. Verify comments against implementation and tests; remove stale claims. Require declaration-level JSDoc for static variables and named constants explaining what they represent and their purpose; apply checklist C6 and the detailed commenting rules even when the name seems clear. Preserve the user-required step prefixes: `1.`, `2.`, and `3.`. Enforce their responsibility boundaries: `1.x` validates, defaults, and normalizes arguments; `2.x` performs business logic, including state-dependent checks and validation of newly received external data; `3.x` produces the final output. Classify statements by their actual work, not their position or comment wording. Use numbered phase comments for multi-step routines. When a phase contains multiple distinct operations, require numbered substep comments (`2.1`, `2.2`, `2.3`, for example) for each operation; one broad `2.` heading is insufficient. Separate substeps with blank lines and do not remove their markers as redundant style. See the reference for early returns, asynchronous work, and cleanup. Apply the required [logical-block spacing rules](references/logical-block-spacing.md): separate major phases and distinct substeps with a blank line, while keeping each cohesive group together. Comment and spacing edits must preserve executable behavior and compiler/linter directives.

## Required architectural readability review

Apply [architectural-readability.md](references/architectural-readability.md) when changes involve orchestration, runtime dispatch, asynchronous state, race tests, or documentation examples. Require auditable state transitions and dispatch coverage, visible effect ordering, deterministic coordination for ordering tests, and accurate language-tagged examples. These checks refine existing cohesion, type, ownership, and documentation rules; they do not mandate a state-machine library, framework, arbitrary file-length cap, or editor theme. Complete checklist items A1–A5 where applicable.

## Required verification and repair loop

After making fixes in the default repair mode, inspect the relevant project's `package.json`, package-manager declaration, lockfile, and repository guidance. Use its existing format, lint, and test scripts with the established package manager and working directory. Check documented equivalents when a script has another name. In a workspace, use its prescribed root orchestration or affected-package commands. Do not invent scripts or install a new toolchain solely to satisfy this loop.

1. **Format:** Execute the project's format command if supported. Inspect any changes it makes. If it fails, diagnose and fix the cause, then restart at step 1. A formatting check does not replace the project's formatting command when a formatter is available.
2. **Lint:** Once formatting succeeds (or is unsupported), execute the project's lint command if supported. Fix lint failures, then return to step 1. Do not proceed to tests while a supported formatting or lint step still fails. Lint autofixes that modify files also require a restart from step 1.
3. **Tests:** Once the earlier supported steps pass, execute the project's test command if supported, using its documented non-watch/CI mode when necessary. If tests fail, diagnose and fix the cause, then return to step 1 so formatting and lint are rechecked before testing again.
4. **Checklist recheck:** After supported commands pass, re-read the final scoped changes and evaluate every checklist ID again, including previously passing and not-applicable items. Reassess applicability after refactoring; record current evidence rather than copying earlier statuses. Any new or remaining failure returns to repair and step 1.
5. **Finish:** Stop only after the supported steps pass in order against the final files and every applicable checklist item passes the final manual recheck. If a later fix, typecheck, build, or integration check changes those files, restart at step 1. Report the actual commands and results, plus unsupported or blocked steps. Unsupported commands are skipped explicitly, never reported as passed.

Continue this repair loop until the failures are resolved; do not stop after merely reporting a fixable lint or test error. Diagnose between attempts rather than rerunning unchanged commands indefinitely. Preserve assertions and meaningful checks: do not skip failing tests, weaken expectations, disable lint rules, or alter scripts just to obtain a successful exit code.

Respect the authorized scope and existing user changes. Investigate failures outside the edited code rather than assuming they predate the work. If resolving a failure requires an unrelated behavior change or unavailable external resources, identify the evidence and concrete blocker, state that verification is incomplete, and request only the missing information or authorization. A missing service or credential is not a passed test and cannot be repaired by retrying forever.

This loop applies to implementation and repair work. An explicit review-only request remains read-only: do not run a mutating formatter or make fixes. A general invocation without that restriction uses the default repair mode. A teaching-only snippet need not acquire a new project scaffold. These workflow requirements are user-defined verification policy, not claims about commands prescribed by Code Complete.

For example, in a pnpm project exposing `format`, `lint`, and a one-shot `test` script, a test repair requires the full sequence again:

```text
pnpm run format → pnpm run lint → pnpm run test
                                      ↓ failure
                                  diagnose and fix
                                      ↓
pnpm run format → pnpm run lint → pnpm run test → all pass
```

## Review output

The initial and final item-by-item checklist reports are mandatory. Show every ID from the checklist, its status, and concrete evidence or a reason for not-applicable/blocked status. Preserve fail → pass transitions so the user can see what was repaired. Never replace the table with “all checks evaluated” or a category summary. Report command results separately and distinguish passed, failed, skipped/unsupported, and blocked checks. A scoped task must label not-applicable items honestly rather than claiming a full-module audit. See the reference for the required report lifecycle.

For each material finding give: severity, location, the reader's specific difficulty, its maintenance consequence, the proposed repair, and the behavior that must remain unchanged. Distinguish a defect from an optional style preference. When useful, show a small TypeScript before/after pair; do not rewrite an entire file merely to illustrate one finding.

Review completion requires evidence, not a numeric readability score. Long conditions, large functions, and many parameters are inspection signals—not automatic failures. Nesting beyond three control-flow levels is an explicit mandatory failure under this skill. Do not invent universal line counts, maximum name lengths, mandatory comment density, a ban on `unknown`, or a requirement that every function be generic.
