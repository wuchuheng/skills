# Required exception-handling review

Apply this reference to failure paths in the requested scope. These are this skill's requirements, informed by *Code Complete* §§8.3–8.4 and adapted to TypeScript. They are not a mandate to replace an established project's entire error strategy. See [research.md](research.md) for attribution and limitations.

## Requirements

1. **Define the failure contract.** Distinguish expected domain outcomes, cancellation, invalid external input, failed operations, and programming defects. Prefer an ordinary return value or discriminated result for expected alternatives when it fits the API. Honor existing rejected-Promise/library conventions; “exceptional” does not simply mean statistically rare. Do not force every network error into a new Result framework.
2. **Catch for a reason.** A catch must recover safely, translate at an abstraction boundary, attach useful context, report at the responsible terminal boundary, or perform required cleanup. Otherwise let the failure propagate. Limit the try block to the operation whose failures are being classified. Do not mistake logging for recovery.
3. **Do not hide failure as success.** No empty catch or fallback such as `[]`, `null`, or `true` unless the contract explicitly permits that outcome for the specific failure. Intentional best-effort suppression must explain why failure cannot affect correctness. Catching every error as “not found” is not a valid recovery policy.
4. **Use trustworthy error information.** New throws should normally use `Error` or the project's established error type. Treat caught values as `unknown`; libraries and JavaScript can throw non-Errors. Use documented types/codes or a validated boundary payload for classification, not arbitrary message substrings or an unchecked cast. An `instanceof` test is useful locally but not a portable wire protocol across realms.
5. **Match the abstraction and retain useful causes.** Pass through the original failure when no translation is needed. When translating, use a meaningful domain/adapter error and preserve the original as a cause when safe and supported. Avoid wrapping at every layer. Validate and sanitize errors crossing Worker/API/UI boundaries; do not serialize internal causes, stacks, credentials, or raw content indiscriminately.
6. **Own diagnostics.** Identify the layer responsible for reporting an unhandled operation failure. Avoid duplicating the same report at every layer before rethrowing. Use safe context such as operation and error code; observability should not replace or mask the application failure. Reuse the existing reporter when appropriate; a centralized framework is optional.
7. **Guarantee cleanup according to the contract.** Release each acquired resource on success and failure, including partial initialization. Await asynchronous cleanup. If operation and cleanup can both fail, specify whether to preserve the primary failure, combine both, or use an established cleanup policy. `finally` can mask the original error; do not return from it. Cleanup cannot undo already committed external work: rollback, compensation, or partial success require an explicit contract.
8. **Own asynchronous failures.** Await or return work whose completion belongs to the caller. Intentionally detached work needs a rejection handler; `void promise` alone does not handle rejection. The reporting callback can also fail, so define its policy. Global unhandled-rejection handlers are a last boundary, not normal per-operation control flow.
9. **Classify cancellation and retries explicitly.** Only recognized cancellation for the owned operation may become a cancellation outcome. Do not swallow an unrelated error merely because a signal is now aborted. Retry only when authorized by the operation's policy, limited in attempts/deadline, cancellation-aware, and safe against duplicated effects. No blanket retries for validation errors or programming defects.
10. **Verify the failure paths affected by a change.** Check success, recognized recoverable failure, unexpected failure, non-Error throws where relevant, operation/cleanup dual failure, and cancellation or detached rejection if present. Preserve error identity, response ordering, and caller-visible behavior during a pure refactor. Any changed failure policy is a bug fix or API change and must be in scope.

## Choosing a response

| Situation | Reasonable response |
| --- | --- |
| A lookup legitimately has no match | Return the API's absence/result representation. |
| A documented optional file is missing | Recover only from the documented missing-file code. |
| A dependency fails and this layer cannot recover | Propagate, or translate once at the boundary. |
| A local invariant is broken | Fail visibly; do not invent a successful fallback. |
| An owned operation is explicitly cancelled | Use its documented cancellation outcome. |
| A best-effort diagnostic fails | Suppress only under a stated noncritical reporting policy. |
| Cleanup fails after the main operation fails | Apply the stated precedence/aggregation policy. |

## TypeScript before-and-after examples

Each block is independent and uses original code. The `Before`/`After` suffixes are comparison labels. These examples illustrate explicit failure-policy changes, not a license to make them during unrelated cleanup.

### 1. Expected absence should not depend on throwing and catching

**Change:** API/model change: a discriminated result replaces a thrown absence. Migrate callers deliberately. Unexpected faults are not converted to absence.

```ts
type Item = { id: number };
type LookupResult = { found: true; item: Item } | { found: false };

export function lookupBefore(items: readonly Item[], id: number): Item {
  const item = items.find(candidate => candidate.id === id);
  if (!item) throw new Error('Not found');
  return item;
}

export function lookupAfter(items: readonly Item[], id: number): LookupResult {
  const item = items.find(candidate => candidate.id === id);
  if (!item) return { found: false };
  return { found: true, item };
}
```

**Verify:** Present and absent identifiers; callers explicitly handle both variants. Keeping a throwing lookup is also valid if that is the established API contract.

### 2. Recover only from the documented optional-file failure

**Change:** Bug fix: permission and unexpected failures no longer masquerade as absent content. Assumes the injected adapter documents `code: 'ENOENT'` for absence.

```ts
export async function readBefore(read: () => Promise<string>): Promise<string | null> {
  try {
    return await read();
  } catch {
    return null;
  }
}

export async function readAfter(read: () => Promise<string>): Promise<string | null> {
  try {
    return await read();
  } catch (error: unknown) {
    const isMissingFile = typeof error === 'object' && error !== null &&
      'code' in error && error.code === 'ENOENT';
    if (isMissingFile) return null;
    throw error;
  }
}
```

**Verify:** Successful read, missing file, denied access, and a thrown string. Unexpected failures preserve identity. Do not apply this property check to arbitrary untrusted transport objects without boundary validation.

### 3. Translate once and preserve the internal cause

**Change:** Error-contract repair: callers receive useful operation context instead of a context-free replacement. This is an in-process error, not a public wire payload.

```ts
export async function saveBefore(write: () => Promise<void>): Promise<void> {
  try {
    await write();
  } catch {
    throw new Error('Failed');
  }
}

export async function saveAfter(write: () => Promise<void>): Promise<void> {
  try {
    await write();
  } catch (cause: unknown) {
    throw new Error('Unable to save draft', { cause });
  }
}
```

**Verify:** Success and rejection; the new error retains the exact original cause. At a UI or RPC boundary, send an allowlisted code/message instead of this raw error. If this layer adds no useful abstraction or context, remove the redundant catch and propagate instead.

### 4. Preserve both operation and cleanup failures

**Change:** Explicit error-policy change: the after version combines dual failures. On a single failure, it preserves the exact original thrown value. ES2022 error APIs are assumed.

```ts
type Lease = { release(): Promise<void> };
type Acquire = () => Promise<Lease>;

export async function runBefore<T>(acquire: Acquire, work: () => Promise<T>): Promise<T> {
  const lease = await acquire();
  try {
    return await work();
  } finally {
    await lease.release();
  }
}

export async function runAfter<T>(acquire: Acquire, work: () => Promise<T>): Promise<T> {
  const lease = await acquire();
  let result: T;
  try {
    result = await work();
  } catch (operationError: unknown) {
    try {
      await lease.release();
    } catch (cleanupError: unknown) {
      throw new AggregateError(
        [operationError, cleanupError],
        'Operation and cleanup both failed',
      );
    }
    throw operationError;
  }
  await lease.release();
  return result;
}
```

**Verify:** Success, acquisition failure, operation-only failure, cleanup-only failure, and both failures—including `throw undefined`. Release exactly once after successful acquisition. Aggregation is one policy; preserving the primary error and reporting cleanup separately can also be correct if the application specifies it.

### 5. Give intentionally detached work a failure owner

**Change:** Bug fix: a deliberately detached task no longer leaves its failures unobserved. If callers need its result, return/await the Promise instead of detaching it.

```ts
type Report = (code: 'TASK_FAILED') => void;

export function launchBefore(task: () => Promise<void>): void {
  void task();
}

export function launchAfter(task: () => Promise<void>, report: Report): void {
  void (async () => {
    try {
      await task();
    } catch {
      try {
        report('TASK_FAILED');
      } catch {
        // Diagnostics are best-effort here and must not create a second unowned failure.
      }
    }
  })();
}
```

**Verify:** Task success, rejected Promise, synchronous throw, and a reporter that throws. Only the responsible owner reports, and no raw exception text is emitted. This suppression is appropriate only for the stated diagnostic policy, never for required persistence or audit delivery.

### 6. Do not treat all failures as cancellation

**Change:** Bug fix: only a recognized cancellation for this operation returns a cancelled outcome. The injected predicate represents the adapter's documented ownership and cancellation contract.

```ts
type Outcome<T> = { status: 'completed'; value: T } | { status: 'cancelled' };

export async function waitBefore<T>(work: () => Promise<T>): Promise<Outcome<T>> {
  try {
    return { status: 'completed', value: await work() };
  } catch {
    return { status: 'cancelled' };
  }
}

export async function waitAfter<T>(
  work: () => Promise<T>,
  isOwnedCancellation: (error: unknown) => boolean,
): Promise<Outcome<T>> {
  try {
    return { status: 'completed', value: await work() };
  } catch (error: unknown) {
    if (isOwnedCancellation(error)) return { status: 'cancelled' };
    throw error;
  }
}
```

**Verify:** Success, owned cancellation, unrelated rejection, and an aborted signal concurrent with an unrelated failure. Merely testing `signal.aborted` or searching an error message is not an adequate implementation of the predicate.
