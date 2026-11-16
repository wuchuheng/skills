# TypeScript repairs: before and after

These are original examples. Each TypeScript block is self-contained; `Before` and `After` name the two versions for comparison, not recommended production suffixes. Conditions and scope assumptions are part of each example. Links to research and the review criteria are in [research.md](research.md) and [criteria.md](criteria.md).

Contents: 1 conditions; 2 nesting; 3 names/units; 4 variable purpose; 5 cohesion/effects; 6 external unknown; 7 internal types; 8 state model; 9 defaults; 10 tables; 11 async cleanup; 12 comments; 13 test intent.

## 1. Shorten a compound condition without eager side effects

**Problem:** Admission, state, and an effectful permission check are packed together.
**Classification:** Behavior-preserving for the stated typed inputs. Short-circuit order matters.

```ts
type Session = { ready: boolean; closed: boolean };

export function admitBefore(
  session: Session | undefined,
  text: string,
  canSpendQuota: () => boolean,
): boolean {
  return !!session && session.ready && !session.closed &&
    text.trim().length > 0 && canSpendQuota();
}

export function admitAfter(
  session: Session | undefined,
  text: string,
  canSpendQuota: () => boolean,
): boolean {
  if (!session) return false;
  if (!session.ready || session.closed) return false;
  if (text.trim().length === 0) return false;
  return canSpendQuota();
}
```

**Why:** Each rejection can be read independently. Do not move `canSpendQuota()` into an eagerly initialized `hasQuota` variable above the guards.
**Verify:** Absent/not-ready/closed session and blank text never call the quota function; the eligible case calls it exactly once. Simple conditions do not need this expansion.

## 2. Flatten nesting while preserving rejection priority

**Problem:** The ordinary action is buried inside state checks.
**Classification:** Behavior-preserving; missing-user rejection still takes precedence.

```ts
type User = { active: boolean };

export function labelBefore(user: User | null, enabled: boolean): string {
  if (user !== null) {
    if (user.active) {
      if (enabled) {
        return 'Ready';
      } else {
        return 'Disabled';
      }
    } else {
      return 'Inactive';
    }
  } else {
    return 'Missing user';
  }
}

export function labelAfter(user: User | null, enabled: boolean): string {
  if (user === null) return 'Missing user';
  if (!user.active) return 'Inactive';
  if (!enabled) return 'Disabled';
  return 'Ready';
}
```

**Why:** The return policy is visible from top to bottom. If the original had cleanup after the nested block, preserve it before introducing early returns.
**Verify:** All combinations, especially multiple failing prerequisites.

## 3. Name the meaning and the unit

**Problem:** `t`, `n`, and a literal conceal the timestamp unit and expiration policy.
**Classification:** Behavior-preserving. This contract considers the item expired at exactly 30 seconds.

```ts
export function expiredBefore(t: number, n: number): boolean {
  return n - t >= 30000;
}

const SESSION_IDLE_TIMEOUT_MS = 30_000;

export function expiredAfter(lastActivityMs: number, nowMs: number): boolean {
  return nowMs - lastActivityMs >= SESSION_IDLE_TIMEOUT_MS;
}
```

**Why:** Names carry information callers otherwise infer from arithmetic. Do not turn the timeout into configurable policy unless the product needs that change.
**Verify:** Just before, at, and just after the deadline. No ambient clock is introduced.

## 4. Give a variable one purpose

**Problem:** One mutable variable means a total, then a count, then an average.
**Classification:** Behavior-preserving, including empty input producing `NaN` in this existing contract.

```ts
export function averageBefore(values: readonly number[]): number {
  let temp = values.reduce((sum, value) => sum + value, 0);
  const total = temp;
  temp = values.length;
  temp = total / temp;
  return temp;
}

export function averageAfter(values: readonly number[]): number {
  const total = values.reduce((sum, value) => sum + value, 0);
  const count = values.length;
  return total / count;
}
```

**Why:** Each value keeps a stable meaning. Returning zero for empty input would be a separate policy change, not part of this refactor.
**Verify:** Empty, singleton, mixed positive/negative, and fractional inputs.

## 5. Separate a calculation from its effect

**Problem:** The calculation is hidden inside persistence orchestration and parameters have opaque names.
**Classification:** Behavior-preserving; the injected writer is still called once, after calculation.

```ts
type InvoiceLine = Readonly<{ quantity: number; unitPriceCents: number }>;
type WriteTotal = (totalCents: number) => Promise<void>;

export async function saveBefore(xs: readonly InvoiceLine[], fn: WriteTotal): Promise<number> {
  let n = 0;
  for (const x of xs) n += x.quantity * x.unitPriceCents;
  await fn(n);
  return n;
}

function calculateTotalCents(lines: readonly InvoiceLine[]): number {
  let totalCents = 0;
  for (const line of lines) {
    totalCents += line.quantity * line.unitPriceCents;
  }
  return totalCents;
}

export async function saveAfter(
  lines: readonly InvoiceLine[],
  writeTotal: WriteTotal,
): Promise<number> {
  const totalCents = calculateTotalCents(lines);
  await writeTotal(totalCents);
  return totalCents;
}
```

**Why:** The helper expresses a domain operation. No class hierarchy, generic pipeline, or service container is needed. Keep the existing public name if renaming would break callers; these suffixes only distinguish the examples.
**Verify:** Same arithmetic order, writer argument/call count, and propagated writer rejection.

## 6. Keep `unknown` at an external boundary and validate it

**Problem:** A cast promises that unchecked data has a numeric identifier.
**Classification:** Bug fix. The after version deliberately rejects malformed inputs that the before version accepts.

```ts
type Finding = Readonly<{ id: number; text: string }>;

export function parseBefore(payload: unknown): Finding {
  return payload as Finding;
}

export function parseAfter(payload: unknown): Finding {
  if (typeof payload !== 'object' || payload === null) {
    throw new Error('Invalid finding');
  }
  if (!('id' in payload) || typeof payload.id !== 'number') {
    throw new Error('Invalid finding');
  }
  if (!Number.isSafeInteger(payload.id)) throw new Error('Invalid finding');
  if (!('text' in payload) || typeof payload.text !== 'string') {
    throw new Error('Invalid finding');
  }
  return { id: payload.id, text: payload.text };
}
```

**Why:** A boundary parser must establish the type, not conceal uncertainty. The returned object is also a detached projection: extra fields and reference identity are not preserved. Confirm that this is the intended boundary contract. This parser assumes decoded plain data, not hostile proxies/getters.
**Verify:** Valid data, null, a string ID, fractional/unsafe IDs, missing text, and non-string text. Do not replace `unknown` with `any` or a falsely trusted input type.

## 7. Stop carrying `unknown` after validation

**Problem:** Internal code knows the domain but discards its type and casts it back.
**Classification:** Type/API tightening; identical runtime behavior for valid Findings. Migrate typed callers or parse external values first.

```ts
type Finding = Readonly<{ id: number; text: string }>;

export function findingIdBefore(finding: unknown): number {
  return (finding as Finding).id;
}

export function findingIdAfter(finding: Finding): number {
  return finding.id;
}
```

**Why:** The signature states the actual prerequisite. A generic helper is unnecessary here. For truly varying method results, a small type derived from the canonical method contract may be appropriate.
**Verify:** Valid domain callers compile; callers holding unvalidated `unknown` must validate before calling. Runtime validation belongs at the boundary, not in every internal accessor.

## 8. Model mutually exclusive states explicitly

**Problem:** Independent flags and optional data permit contradictory states.
**Classification:** Model/API change. Migrate producers and callers; do not silently choose which contradictory old state wins.

```ts
type Finding = { id: number };
type LooseState = {
  loading: boolean;
  failed: boolean;
  findings?: Finding[];
  error?: string;
};

export function statusBefore(state: LooseState): string {
  if (state.loading) return 'Loading';
  if (state.failed) return state.error ?? 'Unknown failure';
  return `${state.findings?.length ?? 0} findings`;
}

type CheckState =
  | { kind: 'loading' }
  | { kind: 'failed'; error: string }
  | { kind: 'ready'; findings: Finding[] };

export function statusAfter(state: CheckState): string {
  switch (state.kind) {
    case 'loading': return 'Loading';
    case 'failed': return state.error;
    case 'ready': return `${state.findings.length} findings`;
    default: {
      const unexpectedState: never = state;
      throw new Error(`Unsupported state: ${String(unexpectedState)}`);
    }
  }
}
```

**Why:** Required data follows the state. This only fits mutually exclusive states: independent capabilities should remain independent.
**Verify:** Each permitted variant, migration of missing-error/contradictory inputs, and compilation when a new variant is added without handling it.

## 9. Distinguish absent values from valid zero

**Problem:** A truthy default changes an intentional zero into a nonzero limit.
**Classification:** Bug fix under the explicit contract that zero disables retries.

```ts
export function retriesBefore(retryCount?: number): number {
  return retryCount || 3;
}

export function retriesAfter(retryCount?: number): number {
  return retryCount ?? 3;
}
```

**Why:** The operator should express the missing-value policy. Do not apply this mechanically when zero is supposed to trigger the fallback. This change also preserves NaN rather than replacing it; validate invalid numbers separately if the boundary requires it.
**Verify:** Undefined, zero, and a positive count; decide the policy for negative and non-finite numbers explicitly.

## 10. Use a table for uniform mappings, not arbitrary control flow

**Problem:** Repeated comparisons obscure that this is a complete static lookup.
**Classification:** Behavior-preserving for the typed Severity domain; untrusted strings still need boundary validation.

```ts
type Severity = 'error' | 'warning' | 'suggestion';

export function priorityBefore(severity: Severity): number {
  if (severity === 'error') return 3;
  if (severity === 'warning') return 2;
  return 1;
}

const PRIORITY_BY_SEVERITY: Record<Severity, number> = {
  error: 3,
  warning: 2,
  suggestion: 1,
};

export function priorityAfter(severity: Severity): number {
  return PRIORITY_BY_SEVERITY[severity];
}
```

**Why:** Missing keys become a compile-time concern. A three-case switch is also reasonable; this is an optional repair unless the mapping is genuinely difficult to maintain. Do not replace effectful branches with eagerly computed table values.
**Verify:** Every allowed key and validation of external keys. The old fallback for arbitrary invalid strings is not preserved by this example.

## 11. Make asynchronous resource ownership visible

**Problem:** A rejected write skips release.
**Classification:** Bug fix, not behavior-preserving cleanup. The example's release function is specified not to throw.

```ts
type Lease = { release(): void };
type Acquire = () => Promise<Lease>;

export async function writeBefore(acquire: Acquire, write: () => Promise<void>): Promise<void> {
  const lease = await acquire();
  await write();
  lease.release();
}

export async function writeAfter(acquire: Acquire, write: () => Promise<void>): Promise<void> {
  const lease = await acquire();
  try {
    await write();
  } finally {
    lease.release();
  }
}
```

**Why:** Ownership starts after acquisition and ends on either write outcome. If release can throw/reject, explicitly decide whether to preserve, aggregate, or replace the original failure; `finally` alone does not settle that policy. Do not fire-and-forget asynchronous cleanup.
**Verify:** Successful write, rejected write, failed acquisition, exact release count, and the promised rejection value.

## 12. Explain why ordering is necessary

**Problem:** The comment repeats the code without describing its invariant.
**Classification:** Comment-only repair; the stated ordering must be true of the actual protocol.

```ts
export async function respondBefore(
  deliverNotifications: () => Promise<void>,
  sendResponse: () => void,
): Promise<void> {
  // Wait for notifications.
  await deliverNotifications();
  sendResponse();
}

export async function respondAfter(
  deliverNotifications: () => Promise<void>,
  sendResponse: () => void,
): Promise<void> {
  // Separate channels have no shared FIFO: acknowledge events before replying.
  await deliverNotifications();
  sendResponse();
}
```

**Why:** The comment prevents a future maintainer from deleting an apparently unnecessary wait. Do not assert protocol guarantees in comments that the code or tests do not establish.
**Verify:** Event acknowledgment precedes the reply; rejection behavior stays unchanged.

## 13. Name the test scenario and observable outcome

**Problem:** Short names and hidden values force readers to reconstruct the assertion's intent.
**Classification:** Test readability refactor. This dependency-free example uses the same assertion and values in both versions.

```ts
function expired(lastActivityMs: number, nowMs: number): boolean {
  return nowMs - lastActivityMs >= 30_000;
}

export function testBefore(): void {
  const t = 100;
  const n = 30100;
  if (!expired(t, n)) throw new Error('Expected expiry');
}

export function expiresAtTheIdleDeadlineAfter(): void {
  const lastActivityMs = 100;
  const idleDeadlineMs = 30_100;
  const isExpired = expired(lastActivityMs, idleDeadlineMs);
  if (!isExpired) throw new Error('Expected expiry');
}
```

**Why:** Scenario, input roles, and expected behavior are visible. In a test framework, use this scenario as the test title and its normal assertion API. Do not replace a production call with a local imitation while cleaning up an existing test; the local function here only makes the illustration self-contained.
**Verify:** Existing assertions and tested production boundary stay unchanged in the actual repository.
