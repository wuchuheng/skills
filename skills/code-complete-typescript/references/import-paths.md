# Mandatory readable import paths and ordering

## I1. Choose the shorter valid import path

The purpose of import-path refactoring is shorter, readable paths, not maximizing alias usage. Compare the relative and supported alias forms for the same target module, using the characters inside the import quotes. Choose the shorter valid form while preserving public entry points, runtime resolution, and ownership boundaries. Do not abbreviate directory names or introduce opaque aliases solely to win a character count.

### Alias boundary

Keep `@/` mapped exclusively to `src/`. Do not remap it to the repository root, add an `@/tests/` override, or introduce additional aliases such as `@tests/` for this rule. For targets outside `src/`, retain relative imports; external packages and built-ins keep their existing module specifiers. The target location decides applicability: a test importing production code inside `src/` still qualifies for comparison.

For example, from `tests/unit/features/writing/finding-session/session-client.spec.ts`, retain this import (illustrative project fragment):

```ts
import { createFixtureEvents, FixtureEvent } from '../../../../support/finding-session-events';
```

It resolves to `tests/support/finding-session-events.ts`. An `@/tests/support/...` spelling would resolve inside `src/tests/support/`, so it is not a valid alternative. Record this as outside the alias boundary under I1, not as an unresolved shortening task.

### Required comparison before editing

For each import under review, perform these steps before deciding to rewrite or retain it:

1. Resolve the current import to its actual module. For a target inside `src/`, derive relative and `@/` candidates from the importing file. For a target outside `src/`, record the boundary decision above and skip the alias comparison.
2. Verify that both candidates resolve to the same module through the applicable tools and respect the same public boundary. An unsupported alias is only a proposed candidate until its mapping is configured and verified.
3. Compare the lengths of the two valid module specifiers, excluding quote characters. Record both candidate strings, their lengths, and the decision in the working checklist evidence; this can be a compact grouped table for many imports.
4. Use the shorter candidate. If lengths are equal, keep the current import. If only one candidate is valid, keep that valid form and record why the other is unavailable. Do not change a path first and compare afterward.
5. Apply the selected changes, then sort imports and verify resolution using the checks below. A decision to retain the current shorter path counts as a completed check, not unfinished work.

Do not assume that every alias is shorter or that every relative import needs replacement. For targets inside `src/`, this comparison is required even when the final decision is to keep the current path.

- If `@/...` is shorter than the current relative path, use the alias after verifying support.
- If the alias is longer, keep the current relative path; this passes I1 and needs no exception or refactoring task.
- If lengths are equal, keep the existing spelling to avoid churn.
- When reviewing an existing long alias, a shorter valid relative path is also appropriate; aliases have no inherent priority over relative paths.
- External packages, built-ins, and asset/Worker URL patterns keep their required semantics. Never import a private module just because its path is shorter.

For example, from a neighboring file in the same worker-message directory, keep this short form (illustrative host-project fragment):

```ts
import { startSessionServer } from './session-server';
```

Do not expand it to `@/features/writing/finding-session/worker-message/session-server`. Conversely, the deep traversal example below benefits from `@/`. Both decisions satisfy the same rule.

If `@/ → src/` is missing and would shorten scoped imports into `src/`, configure that mapping consistently in default repair mode. Do not introduce other aliases or map `@/` to a different source directory. If the project lacks `src/` or already assigns another meaning to `@/`, preserve its configuration and report the incompatibility rather than silently repointing the alias. Respect explicit read-only instructions and package boundaries.

### Establish actual support before rewriting paths

- Inspect the relevant `package.json`, tsconfig inheritance/references, bundler settings, test runners, lint resolvers, and scripts that execute files directly. Record the paths that actually consume the changed files.
- TypeScript `paths` helps type resolution but does not by itself rewrite emitted import specifiers. A successful editor lookup is not proof that Node, a Worker bundle, or a test runner can resolve the same path.
- Configure every affected execution path using its existing supported mechanism. Prefer the existing stack and configuration over installing an alias plugin or loader unnecessarily. Verify custom builds separately when they bypass the main config, including browser-test harness builds and Worker entry compilation.
- A published library must not leak an application-only alias into emitted JavaScript or declarations that consumers cannot resolve. Preserve package exports and environment-specific public entries. Do not use aliases to bypass architectural import rules.
- Preserve bundler-specific asset/Worker patterns such as `new URL('./worker.ts', import.meta.url)` unless the actual toolchain supports the proposed alternative and its output is verified. They are not ordinary import declarations to rewrite mechanically.
- Migrate the target imports after establishing support. Compare remaining traversals with their alias alternatives; a shorter or equal existing relative path is compliant. Explain only genuine resolution or boundary blockers where the shorter candidate cannot be used. If a resolver cannot support the mapping within current constraints, record the concrete blocker rather than introducing imports that fail at runtime.

Before (illustrative fragment whose dependency comes from the host project):

```ts
import { startSessionServer } from '../../../src/features/writing/finding-session/worker-message/session-server';
```

After, when `@/` resolves to that project's `src/` in every applicable tool:

```ts
import { startSessionServer } from '@/features/writing/finding-session/worker-message/session-server';
```

This private import is appropriate only inside its permitted feature or a test that is allowed to inspect internals. A production host must still import the feature's supported public entry. The shorter path must resolve to the same module as before.

## I2. Sort imports consistently without changing initialization

Use the existing formatter/lint import-order rules first. Do not introduce a conflicting sorter or reconfigure the repository solely for a preferred visual style. Where no convention exists, use these groups, separated by one blank line:

1. Node/runtime built-ins, using their established specifiers.
2. External packages.
3. Source-alias imports.
4. Relative imports, including those retained because they are shorter.

Within a reorder-safe group, sort module specifiers lexicographically and sort named specifiers by imported name. Follow the project's type-import style; use type-only imports for type-only dependencies when supported, without removing an intentionally required runtime initialization import. Preserve aliases and public names.

**Side-effect safety takes precedence over sorting.** Imports can execute modules even when they bind a name. Treat deliberate initialization order, polyfills, registration imports, CSS cascade order, and known cycles as ordering constraints. Preserve the relevant sequence, document why, and sort only independent runs around it. Do not blindly move bare imports to the top, combine imports across an ordering barrier, or rewrite dynamic imports in ways that change lazy loading.

Before (illustrative host-project fragment):

```ts
import { readProfile } from '../../src/profile/read-profile';
import { z } from 'zod';
import { join } from 'node:path';
import type { Profile } from '../../src/profile/types';
```

After, assuming the project already depends on `zod`, has the alias configured, and these modules have no order-sensitive initialization:

```ts
import { join } from 'node:path';

import { z } from 'zod';

import { readProfile } from '@/profile/read-profile';
import type { Profile } from '@/profile/types';
```

Do not install the illustrative dependency or create these sample modules in a target project. Use its actual imports and dependencies.

## Required verification

Record the selected path forms, same-target resolution evidence, and any alias configuration changes in I1. Do not fail a shorter relative import just because an alias exists. For I2, record the active convention and any ordering constraints. Run the required format → lint → tests repair loop, plus typecheck and build/runtime coverage for the execution paths whose resolution changed. A path-only change is still a resolution change; typecheck alone is insufficient when bundling or direct execution is affected. A project with no package scripts uses its available relevant checks and reports unsupported steps explicitly.

For a skill-documentation-only change, verify example assumptions and links without pretending illustrative imports can run independently of the host project.
