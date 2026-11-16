# Architectural readability and executable examples

These are required inspection criteria for applicable changes. They refine the skill's existing requirements rather than imposing new architecture on every module. Their concrete TypeScript techniques are project guidance, not quotations from Code Complete.

## A1. Make state transitions and ownership auditable

When several flags, promises, collections, or callbacks jointly control a lifecycle, identify valid combinations, the owner of each mutation, and the events that change admission or cleanup behavior. A reader must be able to explain what can happen before and after each relevant `await`.

Keep independent facts independent. Prefer a discriminated union when states are mutually exclusive and it removes invalid combinations; do not mechanically replace all booleans with a state machine. A short invariant comment or transition table may suffice. State why a snapshot must be captured before yielding, why a guard must be rechecked afterward, and who retains a failed cleanup outcome. Test the meaningful competing transitions.

For a large factory, evaluate separate cohesive owners for transport, pending calls, and subscriptions. Extract only where the boundary reduces the state and dependencies a reader must track. Do not split a closure into modules that share the same mutable bag, introduce an event bus, or impose a line-count threshold.

## A2. Preserve validation evidence and dispatch coverage

For finite method/event protocols, inspect every representation: public names, runtime allowlists, argument/result validation, dispatch, and adapters. Require a compile-time exhaustiveness check, a typed complete mapping, or a direct coverage test so adding an operation cannot silently bypass a required branch. Use the smallest mechanism that fits the existing design.

Untrusted transport data remains `unknown` until validated. When a boolean validator forces a caller to repeat assertions, consider returning a validated discriminated command or using a suitable type predicate to carry the proof. Existing assertions are acceptable only where their runtime justification is traceable; a type assertion itself is never validation. Reject unknown wire values separately from checking exhaustiveness of already validated domain variants. Avoid complicated generic dispatch frameworks when a typed switch is clearer.

## A3. Keep effects visible in the orchestration

Review nested expressions that combine dispatch, awaiting, validation, mutation, and returned output. Split them into meaningful named results when this makes sequence, ownership, or failure boundaries easier to inspect. A simple forwarding return does not need extra variables. Do not change evaluation order, awaited versus detached work, short-circuit behavior, or Promise identity as a cosmetic repair.

Number blocks according to the existing responsibility rules. An operation hidden inside a `return` expression is still processing; labeling it `3.` does not make dispatch or cleanup result construction. When a routine's primary purpose is validation, its schema decisions are its core processing; do not add circular validation of the validator merely to populate an input phase. The distinction is argument preparation versus the routine's actual job.

Before: dispatch and validation are hidden inside the return expression.

```ts
export async function readBefore(
  invoke: () => Promise<unknown>,
  decode: (value: unknown) => number,
): Promise<number> {
  // 3. Return the result.
  return decode(await invoke());
}
```

After: the call and validation remain in the same order, with output clearly separated. No argument preparation is fabricated.

```ts
export async function readAfter(
  invoke: () => Promise<unknown>,
  decode: (value: unknown) => number,
): Promise<number> {
  // 2.1 Wait for the remote operation to finish.
  const reply = await invoke();

  // 2.2 Validate the received data before exposing it to the caller.
  const result = decode(reply);

  // 3. Return the validated result.
  return result;
}
```

## A4. Prove ordering with signals rather than elapsed time

For a test asserting interleaving, readiness, cancellation, or disposal races, coordinate through an explicit signal, gate, acknowledged event, or an appropriate controlled clock. A fixed sleep does not prove that another participant reached the intended state. Retain finite test timeouts to detect hangs and guarantee fixture cleanup when an assertion fails.

Real timeout behavior may require an elapsed-time test; distinguish testing the deadline itself from using a delay to guess ordering. Fake clocks do not automatically advance Worker messages or external I/O. Preserve real transport integration coverage where it matters.

Before: the test assumes waiting makes the work start. This fragment requires a test harness providing `start` and `assertStarted`.

```ts
export async function observeBefore(
  start: () => void,
  assertStarted: () => void,
): Promise<void> {
  // 2.1 Start the work.
  start();

  // 2.2 Guess that scheduling has completed after a delay.
  await new Promise<void>(resolve => setTimeout(resolve, 50));
  assertStarted();
}
```

After: the harness provides a signal resolved at the actual operation-entry point. This is an intentional test-harness contract change, not a behavior-preserving API rename. Use the test runner's timeout and fixture cleanup to bound failure.

```ts
export async function observeAfter(
  start: () => void,
  started: Promise<void>,
  assertStarted: () => void,
): Promise<void> {
  // 2.1 Start the work whose entry point resolves the supplied signal.
  start();

  // 2.2 Assert only after observing the actual state transition.
  await started;
  assertStarted();
}
```

## A5. Use syntax highlighting accurately and keep examples trustworthy

Use `ts` or `typescript` for TypeScript fences, `tsx` for JSX-bearing TypeScript, `sh` for shell commands, `json` only for valid JSON, `mermaid` for diagrams, and `text` for trees, logs, or pseudocode. Use appropriate source file extensions. Do not depend on editor colors to communicate ownership, branching, or failure: names, types, layout, and comments must remain readable as plain text. Do not install a highlighting dependency or change a user's editor theme just to satisfy this rule.

When changing executable examples, parse/typecheck self-contained snippets where tooling is available. Identify illustrative fragments and their omitted dependencies; do not claim they run standalone. Where before/after examples claim preserved behavior, check outputs and effects or compare emitted code when only layout/comments change. Keep explicit API/behavior changes labeled.

Reconcile affected paths, interfaces, file trees, and lifecycle descriptions with the current implementation. Timestamp verification evidence and state its scope. Preserve historical results as historical; do not silently update old test totals or describe an old failing gate as the current state without rerunning it. A historical log is not evidence of the present worktree.

During review, record any missing rendering/runtime evidence. Valid syntax and a correct fence support highlighting, but do not prove a particular editor or Markdown renderer displayed it correctly.
