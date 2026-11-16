# Blank lines separate logical blocks

## Required rule

Treat a cohesive group of statements as a paragraph. A blank line signals that one thought has finished and the next begins. This reduces the effort needed to find validation, calculations, effects, and results while scanning a routine.

- Put one blank line between the input block (`1.`), processing block (`2.`), and return block (`3.`). Put it before the next step comment, not between that comment and its code.
- Inside a major phase, put one blank line between distinct substeps, including `1.1`, `1.2`, `2.1`, and `2.2`. The same rule applies to unnumbered logical groups and groups inside loops, branches, and async routines.
- Keep tightly related statements together: a declaration and its guard, fields that construct one value, or several assertions about one outcome. Do not insert blank lines after every statement or mechanically after every brace.
- Use numeric prefixes followed directly by meaningful descriptions. Do not add category labels such as `Input:` or `Processing:`.
- Keep comments/JSDoc and compiler directives attached to the statements or declarations they describe. Do not add empty lines immediately inside otherwise compact blocks merely to pad them.
- Identify the boundary by purpose, not by line count. Several lines can form one thought; two single statements can represent distinct thoughts. Add no empty phase or redundant comment just to obtain three blocks.
- Preserve order and behavior. Whitespace does not justify moving side effects, changing short-circuit evaluation, splitting `return` from its expression, or changing template-literal content. If interleaved responsibilities need reordering or extraction, treat that as a separate refactor and establish safety first.

For review, name the responsibilities that are visually merged and show where the missing pause belongs. For repair, group by purpose, add or remove blank lines, retain numbered prefixes, and run formatting checks. A formatter cannot determine every semantic boundary; inspect the logical groups as well. For layout-only changes, compare executable output or syntax where practical.

## Original TypeScript before/after examples

The examples below preserve executable statements and their order. Only whitespace and the illustrative function suffix change. They apply the book's code-paragraph principle; they are not copied book listings.

### 1. Separate input, processing, and return

Without paragraph breaks, the phase comments compete with a continuous wall of statements. Place the pause before each next phase.

Before:

```ts
export function normalizeBefore(text: string): readonly string[] {
  // 1. Normalize surrounding whitespace.
  const normalized = text.trim();
  // 2. Keep distinct tokens in first-occurrence order.
  const tokens = normalized === '' ? [] : normalized.split(/\s+/);
  const uniqueTokens = [...new Set(tokens)];
  // 3. Return the token snapshot.
  return uniqueTokens;
}
```

After:

```ts
export function normalizeAfter(text: string): readonly string[] {
  // 1. Normalize surrounding whitespace.
  const normalized = text.trim();

  // 2. Keep distinct tokens in first-occurrence order.
  const tokens = normalized === '' ? [] : normalized.split(/\s+/);
  const uniqueTokens = [...new Set(tokens)];

  // 3. Return the token snapshot.
  return uniqueTokens;
}
```

### 2. Separate smaller steps within a major phase

Filtering and aggregation are different substeps even though both belong to processing. The same distinction applies to normalization and validation within input handling.

Before:

```ts
export function totalBefore(amounts: readonly number[], fee: number): number {
  // 1.1 Normalize the fee.
  const normalizedFee = Math.max(0, fee);
  // 1.2 Reject a non-finite fee.
  if (!Number.isFinite(normalizedFee)) throw new Error('Invalid fee');
  // 2.1 Select positive amounts.
  const positiveAmounts = amounts.filter(amount => amount > 0);
  // 2.2 Aggregate the selected amounts with the fee.
  const subtotal = positiveAmounts.reduce((sum, amount) => sum + amount, 0);
  const total = subtotal + normalizedFee;
  // 3. Return the calculated total.
  return total;
}
```

After:

```ts
export function totalAfter(amounts: readonly number[], fee: number): number {
  // 1.1 Normalize the fee.
  const normalizedFee = Math.max(0, fee);

  // 1.2 Reject a non-finite fee.
  if (!Number.isFinite(normalizedFee)) throw new Error('Invalid fee');

  // 2.1 Select positive amounts.
  const positiveAmounts = amounts.filter(amount => amount > 0);

  // 2.2 Aggregate the selected amounts with the fee.
  const subtotal = positiveAmounts.reduce((sum, amount) => sum + amount, 0);
  const total = subtotal + normalizedFee;

  // 3. Return the calculated total.
  return total;
}
```

### 3. Keep nested logical groups visible

Inside a loop, checking whether a prepared candidate qualifies for the result and recording it are separate groups. A blank line exposes that boundary without adding a comment to every statement.

Before:

```ts
export function selectBefore(values: readonly string[]): string[] {
  // 1. Normalize the supplied values.
  const normalizedValues = values.map(value => value.trim());

  // 2.1 Prepare independent output storage.
  const selected: string[] = [];
  // 2.2 Collect nonempty normalized values.
  for (const normalized of normalizedValues) {
    if (normalized === '') continue;
    selected.push(normalized);
  }
  // 3. Return the collected values.
  return selected;
}
```

After:

```ts
export function selectAfter(values: readonly string[]): string[] {
  // 1. Normalize the supplied values.
  const normalizedValues = values.map(value => value.trim());

  // 2.1 Prepare independent output storage.
  const selected: string[] = [];

  // 2.2 Collect nonempty normalized values.
  for (const normalized of normalizedValues) {
    if (normalized === '') continue;

    selected.push(normalized);
  }

  // 3. Return the collected values.
  return selected;
}
```

### 4. Repair excessive spacing without losing phase boundaries

Blank lines after each statement obscure which values belong together. Keep related declarations adjacent and keep each comment attached to its block.

Before:

```ts
export function rangeBefore(start: number, length: number): { start: number; end: number } {
  // 1. Normalize the range inputs.

  const normalizedStart = Math.max(0, start);

  const normalizedLength = Math.max(0, length);

  // 2. Calculate the range boundary.

  const end = normalizedStart + normalizedLength;

  // 3. Return the normalized range.

  return { start: normalizedStart, end };
}
```

After:

```ts
export function rangeAfter(start: number, length: number): { start: number; end: number } {
  // 1. Normalize the range inputs.
  const normalizedStart = Math.max(0, start);
  const normalizedLength = Math.max(0, length);

  // 2. Calculate the range boundary.
  const end = normalizedStart + normalizedLength;

  // 3. Return the normalized range.
  return { start: normalizedStart, end };
}
```
