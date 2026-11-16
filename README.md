# Skills

Reusable instructions for AI coding agents, maintained by [wuchuheng](https://github.com/wuchuheng). Each skill provides a focused workflow and supporting examples that an agent can use when reviewing or improving code.

Repository mirrors: [GitHub](https://github.com/wuchuheng/skills) · [Codeberg](https://codeberg.org/wuchuheng/skills).

## Available skills

This repository currently contains one skill. The list below covers every skill included here.

| Skill | Purpose |
| --- | --- |
| [code-complete-typescript](skills/code-complete-typescript/SKILL.md) | Review and improve TypeScript readability and maintainability using practical Code Complete principles. |

### code-complete-typescript

A code review and refactoring guide for making TypeScript easier to understand and maintain. Use it when reviewing a change, simplifying a working module, or learning how to improve code through before/after examples.

- **Readability:** Require meaningful variable/constant names and at most three control-flow nesting levels per routine. Refactor deeper logic using safe guards or cohesive helpers.
- **Imports:** Choose the shorter readable relative or supported alias path; keep the existing path on ties. Keep `@/` mapped exclusively to `src/`; compare paths for targets inside `src/` and retain relative paths outside it. Do not add test aliases or overrides. Configure the missing `@/ → src/` mapping only where it shortens scoped imports, verify affected tools, and sort imports without changing initialization order.
- **Reuse:** Consolidate repeated same-rule logic into cohesive helpers and repeated same-domain literals into canonical constants, value objects, or enums. Track occurrences, migrate consumers, and verify ownership and dependency direction.
- **Types and failures:** Validate untrusted data, use meaningful internal types, and make exception handling and cleanup ownership explicit. Require every catch or rejection handler to recover, propagate, report once at its owner, or justify safe suppression.
- **Architecture:** Make lifecycle transitions, dispatch coverage, and effect ordering auditable; use deterministic coordination in race tests and accurate language-tagged documentation examples.
- **Comments:** Require meaning-and-purpose JSDoc on constants and static variables. Document contracts and intent; enforce correctly classified phase/substep numbers and blank lines between logical blocks.
- **Safe changes:** Distinguish behavior-preserving refactoring from bug fixes and API changes. Run supported project commands in order: format, lint, then tests. Fix failures and restart from formatting until verification passes.

By default, invoking the skill reviews and directly fixes applicable issues, then rechecks the checklist and repeats verification until all applicable items pass. No separate fix approval is needed for ordinary scoped repairs. Explicit review-only or teaching-only requests override that default. Without an explicit target, reviews cover current staged, unstaged, and relevant untracked changes. The [mandatory checklist](skills/code-complete-typescript/references/review-checklist.md) records each requirement as pending, pass, fail, not applicable, or blocked, with evidence. The agent shows an initial item-by-item report, tracks repairs, and presents a complete final table with status transitions and evidence. After repairs and verification commands, every checklist ID is evaluated again against the final scoped changes, including previously passing items and previously inapplicable items. New failures restart repair and verification; completion requires every applicable item to pass with current evidence. Unsupported commands are reported explicitly; blockers prevent an unqualified completion claim. The skill does not commit or push without a separate user request. It is not a general security audit or a formatting-only tool.

Read the [skill instructions](skills/code-complete-typescript/SKILL.md) for its workflow and links to detailed references.

Numbered step comments use `1.x` for argument validation/defaults/normalization, `2.x` for business processing (including business-rule checks), and `3.x` for the final result. Multiple operations within a phase require individual substep comments and blank lines. Write the description directly after the number, without labels such as `Processing:`:

```ts
/**
 * Counts distinct whitespace-delimited tokens without changing their case.
 * @param text - Text whose repeated tokens should count once.
 * @returns The number of distinct tokens, or zero for whitespace-only input.
 */
export function countUniqueTokens(text: string): number {
  // 1. Normalize surrounding whitespace.
  const normalizedText = text.trim();

  // 2.1 Split nonempty text into tokens.
  const tokens = normalizedText === '' ? [] : normalizedText.split(/\s+/);

  // 2.2 Preserve one occurrence of each exact token.
  const uniqueTokens = new Set(tokens);

  // 3. Return the distinct token count.
  return uniqueTokens.size;
}
```

See [names and nesting](skills/code-complete-typescript/references/names-and-nesting.md) for the precise depth-counting rules and [import paths](skills/code-complete-typescript/references/import-paths.md) for alias setup, ordering, and runtime-resolution checks.

This is a practical adaptation of Steve McConnell's *Code Complete*, second edition, with additional TypeScript and author conventions. It is not an official certification or a reproduction of the book.

## Install

You need Node.js and npm, which provides `npx`. The CLI is named **`skills` (plural)**.

From your project directory, install the skill and follow the agent-selection prompts:

```sh
npx skills add wuchuheng/skills --skill code-complete-typescript
```

To install globally for Codex across projects:

```sh
npx skills add wuchuheng/skills --skill code-complete-typescript --agent codex --global
```

To inspect the available skills without installing:

```sh
npx skills add wuchuheng/skills --list
```

The GitHub shorthand above uses this repository. You can also install from the Codeberg mirror:

```sh
npx skills add https://codeberg.org/wuchuheng/skills.git --skill code-complete-typescript
```

See the [official Skills CLI reference](https://github.com/vercel-labs/skills/blob/main/docs/commands.md) for supported agents and installation options.

## Use

After installation, invoke the skill for the default review-and-repair workflow:

```text
Use code-complete-typescript on the current changes.
```

The agent checks staged, unstaged, and relevant untracked files, repairs applicable issues directly, and repeats the checklist and supported verification commands until they pass or a concrete blocker remains. To limit the target, name a file or directory in the request.

To request findings without edits, say so explicitly:

```text
Use code-complete-typescript to review the current TypeScript changes.
Report readability and maintainability findings without editing files.
```

For teaching without project edits:

```text
Use code-complete-typescript to explain how to reduce nested control flow.
Show TypeScript before-and-after examples without changing project files.
```

Default repairs preserve established contracts. Public API redesign, new business requirements, and external publication remain outside that default scope.

## Layout

```text
skills/
└── code-complete-typescript/
    ├── SKILL.md          # Entry point and workflow.
    ├── agents/           # Agent UI metadata.
    └── references/       # Criteria, research, and TypeScript examples.
```

Start with a skill's `SKILL.md`; open its supporting references as needed.
