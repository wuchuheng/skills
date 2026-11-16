# Readability and maintainability criteria

These are this skill's operational criteria, informed by the book and adapted to TypeScript. They are not quotations or a claim that every item is a literal book requirement. Source mapping and research limits are in [research.md](research.md).

## What a reviewer should be able to determine

A reader should be able to identify the operation's purpose, valid input/state, ordinary path, failure policy, effects, and cleanup owner. Require a repair when a demonstrated ambiguity or hidden dependency materially prevents that understanding; treat equivalent stylistic alternatives as optional.

| Area | Observable acceptance criterion | Typical repair / example |
| --- | --- | --- |
| Conditions | The condition expresses a recognizable decision without mentally combining unrelated policies. Evaluation order remains visible; control-flow nesting never exceeds three levels per routine. | Guards for rejection paths; a named predicate for a domain rule; a decision table for a fixed mapping. Examples 1, 2, 10. |
| Names | Every variable and constant name identifies its meaning and role; include units where ambiguous. A function's name does not conceal important effects. | `requestTimeoutMs`, `pendingResponses`, `saveInvoice`; avoid `x`, `temp`, `processData` when their meaning is not local and obvious. Examples 3–5. |
| Routines | A routine has a coherent purpose and consistent abstraction level. A caller need not understand its private storage representation. | Extract a domain calculation from orchestration; avoid splitting every few lines into `step1`, `step2`. Example 5. |
| Variables | Each variable has one meaning, is initialized before use, and has the smallest useful scope. | Distinct names for totals and counts; local `const` values where reassignment is unnecessary. Example 4. |
| Imports | The shorter valid relative/alias path is used, retaining existing spelling on ties; imports are consistently sorted without altering initialization or feature boundaries. | Apply [import-paths.md](import-paths.md); verify compiler, build, and runtime/test resolution. |
| Boundary types | Untrusted values are checked at entry. Once validated, domain code receives a useful type. | `unknown` plus a parser at JSON/Worker boundaries; typed parameters internally. Examples 6, 7. |
| State models | Permitted variants and their required data are explicit; callers do not guess relationships between optional fields. | Discriminated union where states are mutually exclusive. Model/API changes require compatible caller migration. Example 8. |
| Defaults | Missing values are distinguished from valid zero, false, and empty strings according to the contract. | Use nullish defaults only if those falsy values are valid. Example 9 is a bug fix, not neutral cleanup. |
| Effects and dependencies | A function receives the capabilities it needs; mutation and ordering are visible. | Inject a writer at the effect boundary; do not add a container or framework merely for injection. Example 5. |
| Errors and resources | Expected outcomes, exceptional failures, recovery, propagation, reporting, and cleanup have explicit ownership. Rejections cannot silently become success. | Apply the required [exception-handling review](exception-handling.md), including dual failures and detached tasks. Example 11 introduces basic cleanup. |
| Comments and layout | Names and structure explain mechanics; comments explain constraints, surprises, or why an order exists. | Document a deliberate exception or protocol invariant. Apply the [commenting review](commenting.md), including public contracts, test intent, and numbered substep coverage within multi-operation phases. Require [blank lines between logical blocks](logical-block-spacing.md), including major phases and distinct substeps. Use the repository formatter. Example 12. |
| Tests | The test names the triggering situation and observable result; important boundary cases are visible. | Preserve public assertions; test failed guards, empty inputs, invalid payloads, and cleanup where relevant. Example 13. |
| Reuse and domain literals | Repeated same-rule logic and same-domain runtime literals must use a canonical owner, even with only two occurrences. Extracted units remain cohesive and dependency direction remains valid. | Apply [encapsulation and reuse](encapsulation-and-reuse.md); distinguish shared policy from coincidentally equal values. |
| Abstraction cost | Each helper/type reduces what a reader must track; it does not just move a branch elsewhere. | Keep a simple switch or loop when a table, generic type, or wrapper would add indirection. Examples 5, 10. |

Apply the mandatory [names and nesting rules](names-and-nesting.md) when assessing these criteria. A fourth control-flow level requires repair; readable naming is required for every binding in the target scope.

## Choosing a conditional repair

- Use a direct expression for a simple relationship. `start <= end` needs no wrapper solely to remove an operator.
- Use guard clauses when they make rejection or exceptional paths clearer. Confirm that moving a return does not skip required cleanup or notification.
- Name a boolean when it communicates a meaningful proposition and its computation is safe at that point. `condition1` is not an improvement.
- Extract a predicate when it represents a coherent rule, especially if independently tested or reused. Do not scatter a single decision among tiny helpers that require repeated navigation.
- Use a table for uniform data mappings; retain explicit branches for behavior with different effects or ordering.
- Prefer positive predicates when they reduce mental negation. Do not mechanically invert every condition; a clear rejection guard can legitimately use `!`.

## Types: when `unknown` is correct

Use `unknown` for data whose structure has not been established: decoded JSON, Worker messages, plugin responses, and caught values. Validate before dereferencing. A type predicate must check the properties it promises; casting is not validation. Boundary validators themselves may accept `unknown`.

Use a concrete domain type inside validated workflows. Preserve relationships between a method and its result with simple generics only when that relationship matters. Avoid replacing straightforward types with deeply nested conditional types just to eliminate all assertions. TypeScript's compile-time guarantees do not constrain runtime senders.

When deliberately malformed values appear in a test, an assertion or `unknown` may be necessary to exercise the runtime boundary. Explain that purpose rather than weakening production types.

## Refactoring hazards to check

Check short-circuit behavior, getter reads, mutation/aliasing, iteration order, NaN and empty inputs where relevant, error identity, return values, and async scheduling. `Promise.all` is not equivalent to sequential awaits. `readonly` is not runtime freezing. Extracted helpers can change binding, exception stacks, or evaluation timing. A Boolean flag sometimes expresses a genuine independent capability; do not force unrelated flags into one state enum.

Test readability by asking another reader to predict a normal case and an edge case, and identify where one policy would change. Metrics can point to a hotspot but cannot establish correctness or comprehension on their own.
