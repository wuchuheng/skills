# Mandatory readable names and maximum nesting

These are user-required acceptance rules. A clear comment does not excuse an unreadable binding, and passing tests does not excuse a fourth control-flow nesting level.

## R2. Every binding must be readable

Review variables and constants at their declaration and use sites, including parameters, destructuring aliases, loop indices/items, callback parameters, caught errors, module storage, and static fields. A reader must be able to tell what the value represents without mentally decoding an abbreviation or scrolling through unrelated code.

- Use the domain role: `findingId`, `pendingResponses`, `subscriptionCleanup`, `sourceRevision`. A name such as `data`, `value`, `result`, or `error` is acceptable only where the immediate context makes that role unambiguous; otherwise qualify it.
- Include units where confusion is possible: `timeoutMs`, `priceCents`, `offsetUtf16`. Name collections by their contents and booleans as meaningful propositions, such as `isDisposed` or `hasPendingCalls`.
- Avoid unexplained abbreviations and placeholders such as `tmp`, `obj`, `v`, or unrelated single letters. A loop index must identify what it indexes when that is not immediately clear. Established mathematical notation can remain when its meaning is explicit in that domain; brevity alone neither passes nor fails the rule.
- Prefer one meaning per binding. Rename or split a variable reused for different concepts; avoid shadowing that makes ownership or identity ambiguous.
- Preserve public property/wire names. Use a meaningful local destructuring alias if the external schema has an awkward name. Renaming a serialized key is an API change, not a local readability repair.
- Keep mandatory constant/static JSDoc; it explains meaning and purpose but does not replace a readable name. Do not invent long names solely to meet a character count.

Before:

```ts
export function costBefore(p: number, q: number): number {
  const x = p * q;
  return x;
}
```

After: assumes the established contract measures price in cents. Verify units from callers before choosing the name.

```ts
export function costAfter(unitPriceCents: number, quantity: number): number {
  const totalCents = unitPriceCents * quantity;
  return totalCents;
}
```

## R6. Count control-flow depth consistently

The body of a function/method/callback starts at depth **0**. Count enclosing control constructs, not braces:

- Each loop (`for`, `for...of`, `for...in`, `while`, `do...while`) adds one level to its body.
- Each `if` adds one level to its branches. An `else if` chain is one decision level; a separate nested `if` adds another. Whether a branch uses braces does not change its depth.
- A `switch` adds one level; its `case` and `default` labels do not add a second level. A loop or branch inside a case does add a level.
- A `try`/`catch`/`finally` structure adds one level to each of its sibling bodies. Nested control inside those bodies adds further levels; `catch` is not nested inside its sibling `try` body.
- Conditional expressions (`condition ? a : b`) count as a decision level, including inside already nested statements. Do not replace a forbidden nested `if` with a ternary to conceal the depth.
- Object/type literals, destructuring, ordinary lexical blocks, and the routine's own braces do not count as control-flow levels.
- Each separately executed routine has its own count. Review nested helpers and callbacks independently, and also inspect the enclosing code for readability. Do not evade the limit by wrapping the fourth loop in an immediately invoked function, anonymous callback, or `map`/`forEach` chain that preserves the same tangled traversal. Extraction must establish a meaningful responsibility with explicit dependencies.

Three nested loops are allowed. A fourth loop inside them is a failure even if each loop has only one statement. Measure the deepest path, not the total number of branches or the sum of sibling blocks.

## Required repair process

1. Identify the deepest path and the responsibility of each level. Record the measured maximum depth in the checklist evidence.
2. Flatten rejection paths with guard clauses when safe, or extract a cohesive operation into a named helper. Prefer helpers in the existing owning feature/module and pass the smallest necessary inputs.
3. Preserve iteration and side-effect order, break/continue behavior, exceptions, mutation, closure captures, and cleanup. Do not turn sequential asynchronous work into concurrency or change short-circuit evaluation merely to reduce depth.
4. Recount the original routine and every extracted helper. Recheck names, numbered phases, substeps, spacing, and constant documentation. Run the required verification loop. Splitting into `step1` and `step2` or passing a large mutable service bag is not an acceptable cohesion repair.

### Before and after: fourth-level traversal

Both versions visit the same values in the same order. The input groups batches into pages, rows, and amounts. The helper owns visiting one page, rather than merely hiding the fourth loop.

```ts
type AmountRow = readonly number[];
type AmountPage = readonly AmountRow[];
type AmountBatch = readonly AmountPage[];

export function visitBefore(
  batches: readonly AmountBatch[],
  visitAmount: (amount: number) => void,
): void {
  for (const batch of batches) {              // Depth 1.
    for (const page of batch) {              // Depth 2.
      for (const row of page) {              // Depth 3.
        for (const amount of row) {         // Depth 4: mandatory failure.
          visitAmount(amount);
        }
      }
    }
  }
}

function visitPage(page: AmountPage, visitAmount: (amount: number) => void): void {
  // 2. Visit each row's amounts in their existing order.
  for (const row of page) {                  // Depth 1.
    for (const amount of row) {              // Depth 2.
      visitAmount(amount);
    }
  }
}

export function visitAfter(
  batches: readonly AmountBatch[],
  visitAmount: (amount: number) => void,
): void {
  // 2. Delegate each page's traversal without changing batch or page order.
  for (const batch of batches) {             // Depth 1.
    for (const page of batch) {              // Depth 2.
      visitPage(page, visitAmount);
    }
  }
}
```

The depth comments are explanatory annotations for this example, not required comments on every production brace. Verify empty inputs, nested empty groups, visit order, and immediate stop when the visitor throws. No class, new framework, global helper registry, or shared mutable state is required.
