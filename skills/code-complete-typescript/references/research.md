# Research and attribution

Researched 2026-11-16. Basis: Steve McConnell, *Code Complete*, second edition, Microsoft Press, 2004. Consulted selected relevant sections and publicly available chapter summaries, not a claim to have exhaustively reviewed every chapter. All TypeScript examples in this skill are original illustrations, not translations of the book's listings.

## Book evidence

The [user-supplied PDF](https://khmerbamboo.wordpress.com/wp-content/uploads/2014/09/code-complete-2nd-edition-v413hav.pdf) identifies the edition and author. Use printed section identifiers below; the PDF's viewer page numbers differ from printed pages.

- **7.2–7.4:** Cohesion, meaningful routine names, and complexity guide decomposition. Section 7.4 treats routine length contextually rather than as an absolute small-function rule. [Chapter summary](https://www.oreilly.com/library/view/code-complete-2nd/0735619670/ch07s08.html).
- **8.1, 8.5:** Protect boundaries from invalid inputs and contain errors. Modern TypeScript parsing below is an adaptation.
- **8.3–8.4 (printed pp. 194–203):** Compare error-handling alternatives; recover when the current layer can do so correctly, otherwise propagate at a suitable abstraction. Avoid unexplained empty catches, understand library exceptions, and establish a consistent project policy. Central reporting is an option, not a requirement for a new framework.
- **10.4, 10.8:** Keep variable scope and lifetime limited; give each variable one purpose. [Chapter summary](https://www.oreilly.com/library/view/code-complete-2nd/0735619670/ch10s09.html).
- **11.1–11.2:** Names should communicate specific meaning, including the role of boolean and status variables. [Chapter summary](https://www.oreilly.com/library/view/code-complete-2nd/0735619670/ch11s08.html).
- **18; 19.1, 19.4:** Tables can express uniform mappings; named partial tests and reduced nesting can clarify control flow. [Control summary](https://www.oreilly.com/library/view/code-complete-2nd/0735619670/ch19s07.html).
- **24.4:** Preserve a recoverable starting point and refactor in small steps. [Section preview](https://www.oreilly.com/library/view/code-complete-2nd/0735619670/ch24s04.html).
- **31, printed pp. 737, 747–748:** Related statements form code paragraphs; blank lines distinguish separate responsibilities and reveal organization. The blank-line reference applies this guidance to original TypeScript examples. The exact numeric prefixes and single-blank-line convention are user/project requirements, not quoted book rules.
- **32.5:** Improve code itself and use comments for intent and surprising constraints, not redundant narration.

[Publisher edition information](https://www.microsoftpressstore.com/store/code-complete-9780735619678) and the [author's book listing](https://stevemcconnell.com/books/) establish provenance. The skill does not reproduce the book or its checklists.

## TypeScript adaptations, not claims from the 2004 book

- `unknown`, type predicates, discriminated unions, and exhaustiveness checks: [TypeScript narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html).
- Meaningful parameter/result contracts and the distinction between `unknown` and `any`: [TypeScript functions](https://www.typescriptlang.org/docs/handbook/2/functions.html).
- `??`, async cleanup, Promise ordering, `readonly` limitations, test-fixture naming, and the concrete review workflow are contemporary implementation guidance. They are not presented as McConnell's TypeScript prescriptions.

The examples intentionally distinguish neutral restructuring from changes to validation, resource guarantees, defaults, or public models. Treat those changes according to the actual application's contract, not as mandatory fixes merely because an example demonstrates them.

## Exception-handling adaptation notes

The dedicated exception reference applies §§8.3–8.4 to TypeScript. The book's constructor/destructor discussion contains C++-specific lifecycle reasoning: do not turn it into a universal ban on throwing from TypeScript factories. Instead require cleanup for partially acquired resources. Its advice to provide diagnostic context must be constrained by current privacy and trust boundaries; do not include credentials or raw user payloads just to make an error detailed.

`Error.cause`, `AggregateError`, Promise ownership, and cancellation classification are modern language/application adaptations, not APIs described by the book. See the official [TypeScript unknown and functions guidance](https://www.typescriptlang.org/docs/handbook/2/functions.html). The new examples explicitly identify changes to error identity, returned types, cleanup precedence, and recovery behavior.

For `Error.cause` target/library support, see the official [TypeScript 4.6 ES2022 guidance](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-6.html#--target-es2022). Compiler library declarations do not polyfill missing runtime APIs.

## Encapsulation and reuse refinement

The explicit requirement to consider extraction before polishing a block, including blocks sharing parameters, reflects the user's requested review priority. It builds on the previously researched chapter 7 guidance on cohesive routines. It does not impose a minimum occurrence count or automatic extraction. The choice among constants, unions, value objects, and enums is a TypeScript adaptation; the [official enum handbook](https://www.typescriptlang.org/docs/handbook/enums.html#objects-vs-enums) also describes `as const` objects as an alternative to enums. Preserve project conventions and serialized values when choosing a representation.
