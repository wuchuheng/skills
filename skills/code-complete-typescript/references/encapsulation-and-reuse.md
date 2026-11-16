# Encapsulation, reuse, and domain literals

## Required decision process

Before adjusting individual statements in a nontrivial block, identify its inputs, output, effects, and domain purpose. Ask whether a named operation would make its callers clearer and allow one implementation of a shared rule. Passing the same parameter to multiple steps is not a reason to leave the operation inline; nor does shared input alone prove that unrelated steps belong together.

Search the owning feature for an existing helper or canonical definition. Prefer extending or reusing it when the contracts match. For repeated blocks, compare meaning, failure policy, ordering, and likely reasons to change—not just syntax. Prefer extraction when those are shared. If they differ, keep them separate or share only the genuinely common calculation. Do not create a generic helper with mode flags to disguise unrelated responsibilities.

A single-use helper is justified when its name establishes a useful abstraction or isolates difficult logic. A trivial expression may be clearer inline. The requirement is to evaluate encapsulation deliberately, not to meet a function-count target. In a review, explain material missed reuse and give the proposed helper's responsibility; when retaining an apparently duplicated block, briefly explain the different ownership or behavior if that distinction matters.

Keep helpers in the narrowest owning module or feature. Prefer explicit small inputs and outputs; a local closure is reasonable when it intentionally owns session state. Do not move feature policy into a global utilities directory just because two callers exist. Preserve `this` binding, short-circuit behavior, mutation, callback identity, error semantics, and execution order. Extraction does not authorize memoization or replacing sequential effects with `Promise.all`.

## Choosing a literal representation

| Meaning | Consider | Avoid |
| --- | --- | --- |
| One shared policy value or unit conversion | Named constant owned by that policy | Generic `FIVE_THOUSAND` with no meaning |
| A closed set needed only for typing | String literal union | Runtime enum solely to decorate a type |
| Named runtime values plus a closed type | `as const` object and derived union | Separate lists of values and types that drift |
| Existing enum-based convention or useful runtime enum | Explicit string enum, or explicit numeric values when required | Implicit numeric members that change a wire protocol |
| Equal values with different reasons to change | Separate domain constants, or clear local literals | One shared constant coupling unrelated policies |

Constants and enums name values; they do not validate incoming data. Preserve exact serialized values and public type compatibility. Introducing an enum can require caller changes even if its runtime strings are unchanged. Follow existing conventions instead of mixing representations unnecessarily. Avoid new public `const enum` exports without checking toolchain compatibility.

## Original TypeScript examples

Each block is independent. `Before` and `After` are comparison labels. These examples supplement the original examples rather than requiring a rewrite of every similar block.

### 1. Extract a cohesive block even when both callers pass the same data

**Classification:** Behavior-preserving for plain immutable line data. Both callers need the same subtotal policy; the caller-specific decisions remain separate.

```ts
type Line = Readonly<{ quantity: number; unitPriceCents: number }>;

export function labelBefore(lines: readonly Line[]): string {
  let subtotalCents = 0;
  for (const line of lines) subtotalCents += line.quantity * line.unitPriceCents;
  return `${subtotalCents} cents`;
}

export function meetsMinimumBefore(lines: readonly Line[], minimumCents: number): boolean {
  let subtotalCents = 0;
  for (const line of lines) subtotalCents += line.quantity * line.unitPriceCents;
  return subtotalCents >= minimumCents;
}

function calculateSubtotalCents(lines: readonly Line[]): number {
  let subtotalCents = 0;
  for (const line of lines) subtotalCents += line.quantity * line.unitPriceCents;
  return subtotalCents;
}

export function labelAfter(lines: readonly Line[]): string {
  return `${calculateSubtotalCents(lines)} cents`;
}

export function meetsMinimumAfter(lines: readonly Line[], minimumCents: number): boolean {
  return calculateSubtotalCents(lines) >= minimumCents;
}
```

**Why:** Shared input is not the abstraction; the subtotal calculation is. The policy now changes in one place. Verify empty lines, exact minimum, and nonempty totals. Do not cache the result as part of extraction.

### 2. Give repeated related literals one runtime owner and derive their type

**Classification:** Representation refactor; public string values remain unchanged. External strings still need validation.

```ts
type StatusBefore = 'open' | 'accepted' | 'ignored';
export function initialBefore(): StatusBefore { return 'open'; }
export function canAcceptBefore(status: StatusBefore): boolean { return status === 'open'; }

export const FindingStatus = {
  Open: 'open',
  Accepted: 'accepted',
  Ignored: 'ignored',
} as const;
export type FindingStatus = typeof FindingStatus[keyof typeof FindingStatus];

export function initialAfter(): FindingStatus { return FindingStatus.Open; }
export function canAcceptAfter(status: FindingStatus): boolean {
  return status === FindingStatus.Open;
}
```

**Why:** Runtime consumers and the type use the same domain vocabulary. A union alone remains reasonable if named runtime values would add no benefit. Verify all variants and the unchanged serialized strings.

### 3. Use an enum when the project calls for named runtime members

**Classification:** Type/API change, despite preserved wire strings. Migrate callers from raw strings to enum members; do not describe this as an entirely neutral refactor.

```ts
export function encodeBefore(command: 'check' | 'dispose'): string {
  return JSON.stringify({ command });
}

export enum SessionCommand {
  Check = 'check',
  Dispose = 'dispose',
}

export function encodeAfter(command: SessionCommand): string {
  return JSON.stringify({ command });
}
```

**Why:** Explicit members communicate the closed command set without changing encoded values. Verify both commands. An enum is an option, not the mandatory answer for repeated strings; use the preceding object pattern if it better matches the codebase.

### 4. Equal literals do not necessarily represent one policy

**Classification:** Behavior-preserving naming; deliberately retain independent ownership.

```ts
export function deadlinesBefore(startedAtMs: number): readonly number[] {
  return [startedAtMs + 5000, startedAtMs + 5000];
}

const WORKER_STARTUP_TIMEOUT_MS = 5_000;
const NOTIFICATION_VISIBLE_DURATION_MS = 5_000;

export function deadlinesAfter(startedAtMs: number): readonly number[] {
  return [
    startedAtMs + WORKER_STARTUP_TIMEOUT_MS,
    startedAtMs + NOTIFICATION_VISIBLE_DURATION_MS,
  ];
}
```

**Why:** Worker readiness and UI visibility have independent reasons to change. A shared `DEFAULT_TIMEOUT_MS` would falsely couple them. When multiple consumers implement the same startup policy, reuse that policy's constant instead. Verify both existing deadlines remain unchanged.
