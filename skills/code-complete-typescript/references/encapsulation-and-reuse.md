# Encapsulation, reuse, and domain literals

## Required decision process

Before adjusting individual statements in a nontrivial block, identify its inputs, output, effects, and domain purpose. Ask whether a named operation would make its callers clearer and allow one implementation of a shared rule. Passing the same parameter to multiple steps is not a reason to leave the operation inline; nor does shared input alone prove that unrelated steps belong together.

Search the owning feature for an existing helper or canonical definition. Prefer extending or reusing it when the contracts match. For repeated blocks, compare meaning, failure policy, ordering, and likely reasons to change—not just syntax. When those are shared across two or more occurrences, extract or reuse one canonical implementation and migrate the relevant callers. This is required, not an optional recommendation. If they differ, keep them separate or share only the genuinely common calculation. Do not create a generic helper with mode flags to disguise unrelated responsibilities.

A single-use helper is justified when its name establishes a useful abstraction or isolates difficult logic. A trivial expression may be clearer inline. For a single-use block, evaluate encapsulation deliberately without a function-count target. For duplicate same-rule blocks, completing the reuse refactor is mandatory. In a review, explain material missed reuse and give the proposed helper's responsibility; when retaining an apparently duplicated block, briefly explain the different ownership or behavior if that distinction matters.

Keep helpers in the narrowest owning module or feature. Prefer explicit small inputs and outputs; a local closure is reasonable when it intentionally owns session state. Do not move feature policy into a global utilities directory just because two callers exist. Preserve `this` binding, short-circuit behavior, mutation, callback identity, error semantics, and execution order. Extraction does not authorize memoization or replacing sequential effects with `Promise.all`.

## Mandatory duplication inventory and repair

1. Search changed files and their owning feature for identical or structurally repeated blocks and literal values. Include `case 'value'`, equality checks, arrays of allowed values, outgoing payloads, error classifications, and default policies. Search beyond changed hunks when needed to find the existing canonical owner.
2. Record each material candidate: occurrence locations, shared meaning or independent meanings, current owner, proposed owner, and required migration. Two occurrences trigger this check; do not wait for three or for a large block.
3. For the same concept, reuse an existing definition or create one canonical owner. Runtime strings need runtime constants, a value object, or an enum; a repeated union alone is not a repair. Repeated algorithms need a cohesive function with explicit inputs and outputs.
4. Migrate relevant consumers together. Search again for the former literal/block and classify every remaining occurrence. Do not leave one switch on constants and another switch on raw strings without a documented reason. If a necessary migration exceeds an explicitly restricted scope, report that restriction as an unresolved item rather than declaring reuse complete.
5. Verify high cohesion and low coupling: the owner represents one vocabulary or rule; it lives in the narrowest common domain/contract layer; consumers depend on that stable owner, not each other. Do not add a generic global constants file, circular imports, or a helper with flags for unrelated policies. Follow explicit project ownership constraints; request an architecture change only if actually necessary.
6. Preserve observable behavior and run the required verification loop. Verify serialized strings independently, confirm dispatcher coverage, and keep tests that can detect a mistaken canonical value.

A retained duplicate needs an explicit reason and evidence. Independent business meanings must not be merged just because their literal values match. Test expectations may deliberately spell a wire value literally so changing the implementation's constant cannot make a protocol-breaking change pass unnoticed. These are semantic independence, not permission to leave duplicated production vocabulary. Language syntax, fixture-specific input data, and unrelated display text are not automatically one domain constant.

Review-only mode reports required repairs; default repair mode implements them. A checklist entry cannot pass with only “considered reuse” or “an enum might help.”

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

**Why:** Runtime consumers and the type use the same domain vocabulary. A union alone is appropriate for a type-only vocabulary; it is insufficient when multiple runtime consumers repeat the same domain values. Verify all variants and the unchanged serialized strings.

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

### 5. Replace repeated switch literals with one domain vocabulary

**Classification:** Behavior-preserving representation refactor for this internal API. The derived string union preserves the accepted values. Before and after are in one independently compilable example.

```ts
type CommandBefore = 'check' | 'dispose';

export function labelBeforeCommand(command: CommandBefore): string {
  switch (command) {
    case 'check': return 'Check text';
    case 'dispose': return 'Close session';
  }
}

export function endsSessionBefore(command: CommandBefore): boolean {
  switch (command) {
    case 'check': return false;
    case 'dispose': return true;
  }
}

// In a real feature, place this vocabulary in its permitted contract/policy owner.
export const Command = {
  Check: 'check',
  Dispose: 'dispose',
} as const;
export type Command = typeof Command[keyof typeof Command];

export function labelAfterCommand(command: Command): string {
  switch (command) {
    case Command.Check: return 'Check text';
    case Command.Dispose: return 'Close session';
  }
}

export function endsSessionAfter(command: Command): boolean {
  switch (command) {
    case Command.Check: return false;
    case Command.Dispose: return true;
  }
}
```

**Why:** The two switches make different decisions, so combining them into a mode-driven dispatcher would couple unrelated callers. They share only the command vocabulary. Reuse that vocabulary, derive its type, and retain each cohesive decision. The canonical definitions are now the only production spellings of the command values in this example.

**Verify:** Compare both functions for both variants. Independently assert that `Command.Check` serializes as `"check"` and `Command.Dispose` as `"dispose"`; do not derive those expected strings from the same constants under test. Unknown transport strings still require runtime validation.
