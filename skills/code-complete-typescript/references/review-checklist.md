# Mandatory review and repair checklist

Use this checklist for every scoped review or repair. It is an execution gate, not an optional list of suggestions. The detailed rules remain in [criteria.md](criteria.md) and its linked references.

## Tracking rules

Create a working row for every ID below with: **ID, status, target location, evidence or repair task**. Initial status is **pending**. After inspection use **pass**, **fail**, **not applicable**, or **blocked**. Every not-applicable decision needs a concrete reason; lack of inspection is pending, not not-applicable. A pass needs inspected evidence or a check result. A fail needs a location and actionable repair. Distinguish unsupported project scripts from commands that exist but fail to run.

All checks must be evaluated, but applicability does not require manufacturing a construct: code without asynchronous effects does not need new Promise handling, and a tiny expression does not need invented phases. Inspecting every duplication candidate is mandatory. Two or more occurrences of the same rule or domain value require canonical reuse and migration of the relevant consumers; merely considering extraction does not pass. Independent meanings and intentionally independent test expectations require an evidenced exception, not a generic style preference.

| ID | Required check | Evidence needed to close it |
| --- | --- | --- |
| S1 | Establish the requested or default changed-file scope, including staged, unstaged, and relevant untracked changes. | Files/range reviewed and any exclusions; preserve existing user edits. |
| B1 | Establish observable behavior and distinguish refactors, bug fixes, and API/model changes. | Relevant inputs, results, mutation, failure, ordering, and compatibility assumptions. |
| R1 | Conditions and nesting express recognizable decisions without hiding evaluation order. | Inspected conditions; short-circuit and effect counts preserved by any repair. |
| R2 | Names communicate purpose, units, and effects; variables have one role and appropriate scope. | Inspected declarations and call sites; ambiguity repaired where demonstrated. |
| R3 | Repeated same-rule logic has one cohesive implementation reused by its consumers. | Duplication inventory, canonical helper search, migrated call sites, preserved effects/errors, and evidence for any retained independent logic. |
| R4 | Repeated same-domain runtime literals, including switch cases, use a canonical value owner. | Locations inventoried; constants/value objects/enums reused across relevant consumers; remaining raw literals explained; exact wire values verified independently. |
| T1 | Types express established domain knowledge; untrusted values are validated. | Boundary narrowing and internal types inspected; no assertion substituted for validation. |
| T2 | State variants and defaults preserve valid zero, false, empty, and missing values. | Relevant branches and default behavior inspected; semantic changes explicitly identified. |
| E1 | Effects, dependencies, subscriptions, and resource lifetimes have clear owners. | Ordering, cleanup, cancellation, and disposal paths inspected where present. |
| E2 | Exceptions and rejections have correct propagation, recovery, and reporting ownership. | No silent success, masked primary failure, or unowned rejection; safe diagnostics. |
| C1 | Public contracts and non-obvious rules have accurate, useful documentation. | JSDoc, business intent, and directives inspected; stale or redundant claims repaired. |
| C2 | Multi-step routines retain numeric phase prefixes for input, processing, and return. | `1.`, `2.`, `3.` semantics checked without category-label words or invented phases. |
| C3 | Multi-operation phases have meaningful numbered substep comments. | Each distinct operation mapped to `2.1`, `2.2`, etc.; one broad heading is insufficient. |
| C4 | Blank lines separate major phases and distinct substeps while related statements stay together. | Visual inspection, including nested blocks; comments remain attached to their code. |
| C5 | Every numbered block obeys the mandatory responsibility boundaries in [commenting.md](commenting.md). | `1.x` contains argument preparation; `2.x` contains business decisions/effects and checks on newly received data; `3.x` produces the final output. No business lookup disguised as input validation, scattered original-argument defaults in processing, or hidden business effects in return construction. |
| C6 | Static variables and named constants have declaration-level standard comments explaining meaning and purpose. | Inventory module/static state, fixed local policy constants, enums/members, and value catalogs; verify adjacent JSDoc states what each represents and why it exists, with units, ownership, or reset policy where relevant. A clear name alone does not pass. |
| R5 | Reuse preserves high cohesion and low coupling. | One clear responsibility per extracted owner; dependencies flow toward the owning contract/policy; no catch-all utilities, cycles, or mode flags joining unrelated rules. |
| A1 | Stateful orchestration has auditable transitions and ownership. | Valid state combinations, mutation owners, captures across awaits, and terminal admission rules inspected; grouped flags justified or replaced when they hide contradictions. |
| A2 | Finite operation dispatch preserves type evidence and coverage. | Unknown wire variants rejected; every declared operation reaches a handler and validator; assertions have a nearby validation proof; additions cannot silently fall through to a success-shaped default. |
| A3 | Effectful work is visible at the correct abstraction level. | Dispatch, waiting, validation, mutation, and cleanup can be followed without decoding nested return expressions; helpers reduce concepts rather than merely relocating lines. |
| A4 | Ordering and lifecycle tests use deterministic coordination. | Signals, gates, acknowledgments, or controlled clocks establish ordering; elapsed delays are not the sole proof that another operation reached a state. |
| A5 | Documentation and examples are structurally readable and accurate. | Correct language fences and file extensions, runnable examples checked or fragments labeled, current contracts/paths reconciled, historical verification distinguished from current evidence. |
| Q1 | Tests describe intent and preserve meaningful assertions. | Relevant normal/failure behavior and any needed regression coverage inspected. |
| V1 | Supported project format, lint, and test commands pass in order after fixes. | Exact commands/results from the required repair loop; unsupported commands explicitly recorded. |
| V2 | Additional required typecheck/build/integration checks pass for the changed boundary. | Actual results or a concrete reason each check is not applicable or blocked. |
| F1 | Final changes satisfy the applicable checklist and remain within authorized scope. | Fresh diff/status review, rechecked requirements, and unresolved items disclosed. |

## Repair and recheck cycle

1. Inspect the target and classify every checklist item. A concrete violation of an applicable required rule must become a finding; do not downgrade it to optional style merely because tests pass.
2. In the default review-and-repair mode, immediately create and execute repair tasks for failed items, recording affected locations and intended behavior preservation. No separate fix request or approval is needed for ordinary scoped repairs. Explicit review-only or teaching-only instructions override this default.
3. Apply the repairs and run the required format → lint → tests loop. Every corrective edit restarts that loop; preserve existing assertions and checks.
4. Re-read the final code and re-evaluate every applicable checklist item, including comments, numbered substeps, blank lines, and any newly touched routines. Do not mark a task complete merely because a patch was applied or a formatter passed.
5. If the manual recheck finds another violation, fix it and repeat verification. Stop when applicable items pass; if genuinely blocked, report incomplete verification and the precise blocker rather than claiming success or retrying unchanged commands indefinitely.

For explicit review-only work, completion means the applicable requirements have been inspected and findings reported; failed items can remain as findings. For repair work, completion requires no remaining applicable failures or pending checks. A blocked item prevents an unqualified completion claim.

## Example working entries

| ID | Status | Target | Evidence / task |
| --- | --- | --- | --- |
| C3 | fail | `session-client.ts`, subscription routine | A single `2.` heading covers registration, delivery, and cleanup. Add meaningful `2.1`, `2.2`, and `2.3` markers. |
| C4 | fail | Same routine | Separate those logical blocks with blank lines; keep each marker attached to its block. |
| E2 | not applicable | Pure string-normalization helper | No throwing operation, Promise, recovery handler, or owned resource in this helper. |
| V1 | pending | Project package scripts | Discover and run supported commands after the repair; record individual format/lint/test outcomes. |

These are illustrative entries, not a substitute for evaluating every ID against the actual scope. In the final response, summarize completed repairs and verification, and explicitly list failures or blockers; avoid dumping repetitive passing rows when a compact summary is sufficient.
