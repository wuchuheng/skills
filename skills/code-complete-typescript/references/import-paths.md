# Mandatory source aliases and import ordering

## I1. Prefer a supported source alias

Use the project's established source alias for imports that cross source directories or walk up to the source root. Prefer `@/features/...` to `../../../src/features/...` when `@/` maps to `src/`. A short `./sibling` or `./styles.css` import can remain relative for cohesion; do not manufacture an alias for every nearby file. External packages, Node built-ins, and paths outside the mapped source root retain their correct import forms.

If there is no established alias, inspect the source layout and configure `@/` to the actual source directory directly in default repair mode. This includes necessary configuration edits and target import migration, without a separate permission question for ordinary reversible changes. Respect explicit read-only instructions, source ownership, and user restrictions. If `@/` already has another meaning, preserve that contract and use the established suitable alias; do not silently repoint it. In a multi-package workspace, respect package boundaries instead of mapping every package through one application's source directory.

### Establish actual support before rewriting paths

- Inspect the relevant `package.json`, tsconfig inheritance/references, bundler settings, test runners, lint resolvers, and scripts that execute files directly. Record the paths that actually consume the changed files.
- TypeScript `paths` helps type resolution but does not by itself rewrite emitted import specifiers. A successful editor lookup is not proof that Node, a Worker bundle, or a test runner can resolve the same path.
- Configure every affected execution path using its existing supported mechanism. Prefer the existing stack and configuration over installing an alias plugin or loader unnecessarily. Verify custom builds separately when they bypass the main config, including browser-test harness builds and Worker entry compilation.
- A published library must not leak an application-only alias into emitted JavaScript or declarations that consumers cannot resolve. Preserve package exports and environment-specific public entries. Do not use aliases to bypass architectural import rules.
- Preserve bundler-specific asset/Worker patterns such as `new URL('./worker.ts', import.meta.url)` unless the actual toolchain supports the proposed alternative and its output is verified. They are not ordinary import declarations to rewrite mechanically.
- Migrate the target imports after establishing support. Search for remaining deep traversals and explain any that cannot use the alias. If a resolver cannot support the mapping within current constraints, record the concrete blocker rather than introducing imports that fail at runtime.

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
4. Relative imports outside the alias or within the local module.

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

Record the source root, chosen alias, affected resolvers, and remaining exceptions in I1. For I2, record the active convention and any ordering constraints. Run the required format → lint → tests repair loop, plus typecheck and build/runtime coverage for the execution paths whose resolution changed. A path-only change is still a resolution change; typecheck alone is insufficient when bundling or direct execution is affected. A project with no package scripts uses its available relevant checks and reports unsupported steps explicitly.

For a skill-documentation-only change, verify example assumptions and links without pretending illustrative imports can run independently of the host project.
