# Comments that explain contracts and decisions

Adapted from the local `js-ts-commenting` skill and reconciled with this skill's Code Complete approach. The API-documentation and required numeric-phase conventions are project practices, not claims about mandates in the book. This reference is self-contained; loading another skill is unnecessary.

## Required review

- Document public APIs with JSDoc above the declaration: purpose, meaningful parameter constraints/units, return semantics, and actual failure behavior. Use `@param` and `@returns` where applicable; do not repeat TypeScript types in prose. Describe synchronous throws separately from Promise rejection in the return description. Never invent errors or promise stronger guarantees than the implementation provides.
- Explain ownership and effects when callers need them: mutation, registration, cancellation, disposal, callback timing, and resource lifetime. Document non-obvious fields where declared. Keep canonical contract documentation with the contract; adapters explain only their additional guarantees.
- Document complex private algorithms and business rules with their approach, invariant, or reason. Simple private helpers need no ceremonial JSDoc. Prefer a meaningful name or cohesive extraction before adding explanatory prose.
- Comments must agree with current code and tests. Remove stale, misleading, and mechanically redundant comments. Preserve `@ts-expect-error`, lint directives, coverage directives, license notices, and tooling annotations unless their removal is deliberately justified and verified.
- Test names should convey the trigger and observable outcome. Add comments for fixture ownership, deliberately invalid values, race coordination, or why a regression matters. Do not narrate every arrange/act/assert statement.
- Separate logical groups with blank lines and end prose sentences with periods. Let the repository formatter determine layout. Do not put sensitive sample payloads or credentials in comments.

## Required phase prefixes: Input, Processing, Return

The user explicitly requires numeric step prefixes: `1.` means input, `2.` means processing, and `3.` means return. Write the number followed directly by the useful step description; do not insert category labels such as `Input:`, `Processing:`, or `Return:`. Preserve them during reviews and refactoring; do not downgrade them to optional style or delete them under an anti-over-commenting rule. This is a user convention, not a claim that Code Complete mandates numbered comments.

- `1.` Validate, normalize, or prepare inputs and the per-call context needed to consume them. In a factory, describe the supplied dependencies and per-instance state being prepared; do not imply that initialization validates data when it does not.
- `2.` Perform the operation, coordinate effects, and make decisions. Use `2.1`, `2.2`, etc. for multiple processing steps, rather than introducing top-level `4.` or higher.
- `3.` Return or emit the outcome, resolve completion, or hand back an API/cleanup handle. The number identifies the responsibility, not the third statement or a requirement to use exactly three code blocks.

For routines with multiple logical steps (including the source skill's longer-than-ten-lines routines with distinct phases), use these numeric prefixes and keep explanatory text specific. Retain existing numeric prefixes even when a routine becomes shorter. Tiny expressions need no fabricated input/processing blocks; when labeling a return-only routine, use `3.` rather than inventing work. Place labels inside function bodies, never at module scope or on fields.

Use substeps such as `1.1` or `2.1` when a phase needs detail. Early rejection stays with the input guard; do not move returns or restructure control flow to make numbering sequential. For async functions, explain what is awaited and what completion means. Cleanup in `finally` remains unconditional processing and must not be moved or described as success-only return. A returned cleanup function is the factory's return value and can have its own internal phases.

In tests, `1.` prepares fixtures, `2.` exercises behavior and verifies assertions, and `3.` completes the test or returns fixture handles. Do not mislabel assertions as a returned value. Ordinary why-comments and compiler directives remain valid without prefixes; the convention applies to step markers, not every comment.

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

After: the required prefixes identify normalization, processing, and the returned outcome. Publication is an effect in the processing phase.

```ts
export function summaryAfter(input: string, publish: (tokens: readonly string[]) => void): number {
  // 1. Normalize surrounding whitespace without changing token case.
  const normalized = input.trim();

  // 2. Preserve first occurrence order when removing exact duplicate tokens.
  const tokens = normalized === '' ? [] : normalized.split(/\s+/);
  const uniqueTokens = [...new Set(tokens)];

  publish(uniqueTokens);

  // 3. Report the size of the token set that was published.
  return uniqueTokens.length;
}
```

For a comment-only repair, verify executable code has not changed and run relevant formatting/lint checks. If a comment exposes a behavior defect, report or fix it within authorization as a separate behavior change; never make a false contract true through an unannounced logic edit.
