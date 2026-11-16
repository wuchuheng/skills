# Comments that explain contracts and decisions

Adapted from the local `js-ts-commenting` skill and reconciled with this skill's Code Complete approach. The API-documentation and required numeric-phase conventions are project practices, not claims about mandates in the book. This reference is self-contained; loading another skill is unnecessary.

## Required review

- Document public APIs with JSDoc above the declaration: purpose, meaningful parameter constraints/units, return semantics, and actual failure behavior. Use `@param` and `@returns` where applicable; do not repeat TypeScript types in prose. Describe synchronous throws separately from Promise rejection in the return description. Never invent errors or promise stronger guarantees than the implementation provides.
- Explain ownership and effects when callers need them: mutation, registration, cancellation, disposal, callback timing, and resource lifetime. Document non-obvious fields where declared. Keep canonical contract documentation with the contract; adapters explain only their additional guarantees.
- Document complex private algorithms and business rules with their approach, invariant, or reason. Simple private helpers need no ceremonial JSDoc. Prefer a meaningful name or cohesive extraction before adding explanatory prose.
- Comments must agree with current code and tests. Remove stale, misleading, and mechanically redundant comments. Preserve `@ts-expect-error`, lint directives, coverage directives, license notices, and tooling annotations unless their removal is deliberately justified and verified.
- Test names should convey the trigger and observable outcome. Add comments for fixture ownership, deliberately invalid values, race coordination, or why a regression matters. Do not narrate every arrange/act/assert statement.
- Separate major phases and distinct logical substeps with a blank line, as required by [logical-block-spacing.md](logical-block-spacing.md). Keep a step comment attached to its code. End prose sentences with periods. Use the repository formatter without treating formatter success as proof of logical grouping. Do not put sensitive sample payloads or credentials in comments.

## Required phase prefixes: Input, Processing, Return

The user explicitly requires numeric step prefixes: `1.` means input, `2.` means processing, and `3.` means return. Write the number followed directly by the useful step description; do not insert category labels such as `Input:`, `Processing:`, or `Return:`. Preserve them during reviews and refactoring; do not downgrade them to optional style or delete them under an anti-over-commenting rule. This is a user convention, not a claim that Code Complete mandates numbered comments.

| Prefix | Required responsibility | Does not belong here |
| --- | --- | --- |
| `1.` / `1.x` | Establish the argument contract: check supplied types, shape, ranges and argument relationships; apply documented defaults; normalize or parse arguments into usable inputs. | Cache/database lookups, business eligibility decisions, resource acquisition, or unrelated session-state initialization. |
| `2.` / `2.x` | Execute the operation using prepared inputs: calculate, query state, check business rules, transform domain data, mutate state, coordinate effects, and release owned resources. Validate newly received external data at the point it enters this processing. | Scattered revalidation or defaulting of the original arguments that should already have been prepared in `1.x`. |
| `3.` / `3.x` | Construct the final output representation and return it, resolve completion, or hand back the promised API/cleanup handle. | New business decisions, persistence, unrelated notifications, or cleanup hidden as result construction. |

These boundaries are mandatory and apply to every substep, including nested numbering. Inspect what the statements do, not merely whether the comment contains a number. A wrong responsibility assignment is a checklist failure.

### Boundary decisions reviewers must enforce

- In a dedicated parser or validator, parsing/schema evaluation is the routine's core `2.x` work; `1.x` is only any prerequisite argument preparation. Classify relative to the routine's purpose rather than assigning every check to input handling.
- Checking that `findingId` is a safe integer is `1.x`; checking whether that finding exists in the session or may be removed is `2.x`.
- A default for a caller-supplied argument is `1.x`. A fallback selected from business state or an external response is a `2.x` policy decision, not an argument default.
- Validating a response after `await` is `2.x`, next to the received data. Never hoist that check to `1.x` before the data exists or omit it because processing supposedly cannot validate anything.
- Factory dependency validation is `1.x`; allocating private session state, attaching listeners, and acquiring resources are `2.x`; returning the API or disposer is `3.x`. A zero-argument factory may have no input phase. Do not label all setup as input handling.
- Publishing a domain event is normally a `2.x` effect. Emission belongs in `3.x` only when it is itself the routine's promised output delivery, such as a response adapter sending its final result.
- Early returns and throws stay beside the guard or business decision that requires them. Do not move code or change ordering just to make the labels increase monotonically. Default parameter expressions already prepare inputs at the signature; do not duplicate them in the body merely to add a `1.x` marker.
- In tests, fixture values representing the operation's inputs can be prepared under `1.x`. Starting a Worker, exercising the operation, assertions, and cleanup are `2.x`. Returning fixture handles or completing the test is `3.x`.

For routines with multiple logical steps (including the source skill's longer-than-ten-lines routines with distinct phases), use these numeric prefixes and keep explanatory text specific. Retain existing numeric prefixes even when a routine becomes shorter. Tiny expressions need no fabricated input/processing blocks; when labeling a return-only routine, use `3.` rather than inventing work. Place labels inside function bodies, never at module scope or on fields.

When a phase contains multiple distinct operations, each operation must have a numbered substep comment such as `1.1`, `1.2`, or `2.1`, `2.2`, `2.3`. A single major-phase heading does not satisfy this requirement for a multi-operation block. Early rejection stays with the input guard; do not move returns or restructure control flow to make numbering sequential. For async functions, explain what is awaited and what completion means. Cleanup in `finally` remains unconditional processing and must not be moved or described as success-only return. A returned cleanup function is the factory's return value and can have its own internal phases.

In tests, follow the same responsibility boundaries above; do not mislabel assertions as a returned value or resource acquisition as argument preparation. Ordinary why-comments and compiler directives remain valid without prefixes; the convention applies to step markers, not every comment.

## Check substep coverage inside major phases

Do not stop reviewing after finding `1.`, `2.`, and `3.`. Read the code inside each phase and identify changes in purpose: parsing, filtering, reconciliation, dispatch, acknowledgment, and cleanup can be separate operations. Every distinct operation in a multi-operation phase needs a meaningful numbered marker immediately before its logical block, with a blank line separating it from the preceding block.

Use sibling numbering in order (`2.1`, `2.2`, `2.3`); use `2.2.1` and `2.2.2` only when a substep itself contains distinct operations. A parent `2.` summary may introduce its children, but need not repeat their descriptions. Nested function bodies describe their own execution and restart their own phase numbering rather than inheriting the enclosing factory's step numbers.

A substep can be short if its responsibility is distinct. Conversely, do not number each assignment or each line of a cohesive calculation. Write what the operation achieves or which invariant it protects, not “Do step 2.1.” Comments and blank lines work together: spacing alone does not replace required substep comments, and numbered comments without spacing do not provide the required separation.

If numbering exposes a large operation with unrelated responsibilities, also consider cohesive function extraction under the reuse rules. Do not use many numbered comments to excuse excessive complexity. Preserve execution order and all failure paths when repairing comments; extraction is a separate refactoring decision.

## Original TypeScript before/after examples

Each pair preserves runtime behavior. These examples illustrate documentation decisions rather than universal required comment counts.

### 1. Document an asynchronous API's ownership and failures

Before: the signature hides who owns disposal and how failures reach callers.

```ts
export function removeBefore(id: number, invoke: (id: number) => Promise<void>): Promise<void> {
  return invoke(id);
}
```

After: assume the injected operation follows the stated contract; verify that contract before copying this documentation.

```ts
/**
 * Requests removal from the current session without disposing the session.
 * @param id - Session-local finding identifier.
 * @param invoke - Bridge operation that resolves after the handler completes.
 * @returns Completion with no payload; rejects if the bridge or handler fails.
 */
export function removeAfter(id: number, invoke: (id: number) => Promise<void>): Promise<void> {
  return invoke(id);
}
```

### 2. Explain protocol ordering instead of narrating an await

Before:

```ts
export async function deliverBefore(deliver: () => Promise<void>, respond: () => void): Promise<void> {
  // Await delivery.
  await deliver();
  respond();
}
```

After: assumes delivery acknowledgment is the protocol's ordering barrier; rejection still prevents the response.

```ts
export async function deliverAfter(deliver: () => Promise<void>, respond: () => void): Promise<void> {
  // Callers must observe the event before the command's completion response.
  await deliver();
  respond();
}
```

### 3. Prefer a meaningful name over a redundant comment

Before:

```ts
export function durationBefore(n: number): number {
  // Multiply seconds by 1000.
  return n * 1000;
}
```

After: this small helper needs no fabricated input or processing phases. The rename below concerns a local parameter, not the public function's identity.

```ts
export function durationAfter(durationSeconds: number): number {
  return durationSeconds * 1000;
}
```

### 4. Preserve the required semantic step prefixes

Before:

```ts
export function summaryBefore(input: string, publish: (tokens: readonly string[]) => void): number {
  const normalized = input.trim();
  const tokens = normalized === '' ? [] : normalized.split(/\s+/);
  const uniqueTokens = [...new Set(tokens)];
  publish(uniqueTokens);
  return uniqueTokens.length;
}
```

After: tokenization, deduplication, and publication are three distinct processing operations. Each receives a numbered substep and a preceding blank line. A single `2.` comment over the entire processing block would leave these operations insufficiently documented.

```ts
export function summaryAfter(input: string, publish: (tokens: readonly string[]) => void): number {
  // 1. Normalize surrounding whitespace without changing token case.
  const normalized = input.trim();

  // 2.1 Split nonempty text into whitespace-delimited tokens.
  const tokens = normalized === '' ? [] : normalized.split(/\s+/);

  // 2.2 Remove exact duplicates while preserving first occurrence order.
  const uniqueTokens = [...new Set(tokens)];

  // 2.3 Publish the token snapshot before reporting completion.
  publish(uniqueTokens);

  // 3. Report the size of the token set that was published.
  return uniqueTokens.length;
}
```

For a comment-only repair, verify executable code has not changed and run relevant formatting/lint checks. If a comment exposes a behavior defect, report or fix it within authorization as a separate behavior change; never make a false contract true through an unannounced logic edit.

### 5. Separate argument checks from business-rule checks

Before: the labels incorrectly place a cache lookup in input handling and an argument check in processing.

```ts
export function removeFindingBefore(id: number, cache: Map<number, string>): void {
  // 1. Find the entry in the current session.
  if (!Number.isSafeInteger(id)) throw new Error('Invalid finding ID');
  const exists = cache.has(id);

  // 2. Validate the supplied identifier.
  if (exists) cache.delete(id);

  // 3. Complete without a result payload.
  return;
}
```

After: argument validity, state-dependent eligibility, and mutation have separate responsibilities. Only comments and whitespace change; the lookup remains after validation.

```ts
export function removeFindingAfter(id: number, cache: Map<number, string>): void {
  // 1.1 Require a safe integer identifier.
  if (!Number.isSafeInteger(id)) throw new Error('Invalid finding ID');

  // 2.1 Determine whether the current session contains the finding.
  const exists = cache.has(id);

  // 2.2 Remove the finding only when it exists.
  if (exists) cache.delete(id);

  // 3. Complete without a result payload.
  return;
}
```
