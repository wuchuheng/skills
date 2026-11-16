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

## Mandatory catch and rejection-handler audit (E3)

The standard policy is: **recover locally when correct; otherwise propagate; report once at the owning boundary.** Inspect `catch` blocks, `.catch(...)`, and the rejection callback of `.then(success, failure)`. Empty handlers, comments-only handlers, early returns, ignored error parameters, and successful fallback values are candidates for inspection. A nonempty handler is not automatically correct.

For each handler, record its purpose and the implementation evidence supporting one of these outcomes:

- **Recover:** Perform a contractually valid fallback or restore a usable state for the recognized failure. Decide whether the degradation needs a warning under the project's policy.
- **Propagate:** Preserve or safely translate the failure for the caller. Remove a catch that only rethrows unchanged when it serves no other purpose. A cleanup-only catch must still propagate; prefer `finally` when appropriate without changing failure precedence.
- **Report at the owner:** An unhandled failure reaching an application boundary or detached task must use the existing reporter, unless a specific safe-suppression policy applies. For awaited operations, retain an explicit failure result or rejection after reporting; logging alone must not turn failure into success. A detached rejection has no awaiting caller, so its reporting handler intentionally terminates that rejection path.
- **Intentionally suppress:** Add a precise comment identifying the recognized expected failure, why ignoring it is safe, and any remaining cleanup/reporting owner. Verify those claims against code and tests. “Ignore error,” “best effort,” and “already handled” without a concrete owner or invariant do not pass. Suppress only the recognized condition; propagate or report other failures. A logger's noncritical failure can be a broader intentional suppression if that policy is explicit and protects the main operation.

Use warning severity for recoverable degradation and error severity for a failed operation, subject to the project's established severity policy. Do not add `console.error` to every catch or log-and-rethrow at every layer. A local log plus propagation requires a distinct documented diagnostic purpose; otherwise leave reporting to the owner. Never log raw private payloads, credentials, or arbitrary caught objects by default.

A reporting function can throw or reject. Use an existing nonthrowing reporting adapter when available, or implement its documented failure policy. Do not recursively send logger failures back through the same logger. Preserve a primary operation error if cleanup or reporting also fails; report or retain the secondary failure according to the established contract.

If the implementation already propagates correctly, a local log is unnecessary. If a failure is swallowed without recovery or reporting, adding a comment is not sufficient unless evidence proves suppression safe. Repair the failure path in default repair mode, classify any changed outcome as a correctness/API change, and verify the affected scenario. Do not change intentional business behavior to satisfy the checklist.

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

### 7. Propagate to the reporting owner instead of silently succeeding

**Change:** Correctness repair: a failed save now rejects instead of resolving successfully. The caller is the established reporting owner; no duplicate local log is added.

```ts
export async function saveBefore(save: () => Promise<void>): Promise<void> {
  try {
    await save();
  } catch {}
}

export async function saveAfter(save: () => Promise<void>): Promise<void> {
  // 2. Preserve save failures for the awaiting caller that owns reporting.
  await save();
}
```

Verify successful completion and preservation of the original rejection. Confirm the actual caller observes the rejection; if the task is detached, use the reporting pattern in example 5 instead.

### 8. Limit documented suppression to the recognized harmless condition

**Change:** Correctness repair: unexpected deletion failures now propagate. In this example, the supplied adapter uses the documented `ENOENT` code when the optional file is already absent. The optional-file contract permits that outcome.

```ts
export async function deleteBefore(remove: () => Promise<void>): Promise<void> {
  try {
    await remove();
  } catch {
    // Ignore deletion errors.
  }
}

export async function deleteAfter(remove: () => Promise<void>): Promise<void> {
  // 2. Attempt deletion and classify only the adapter's expected absence failure.
  try {
    await remove();
  } catch (error: unknown) {
    const alreadyAbsent =
      typeof error === 'object' && error !== null &&
      'code' in error && error.code === 'ENOENT';
    if (!alreadyAbsent) throw error;

    // The optional file is already absent, so the required final state is satisfied.
    // No recovery or warning is needed; all other failures propagate to the caller.
  }
}
```

Verify success, expected absence, an unexpected code, and a non-Error rejection. In application code, reuse an existing canonical adapter classifier rather than copying this example's inline classifier into several callers.
