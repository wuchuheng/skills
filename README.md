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

- **Readability:** Improve names, simplify conditions, and organize cohesive functions.
- **Reuse:** Evaluate whether logic should become a shared function and whether repeated domain literals should become constants, value objects, or enums.
- **Types and failures:** Validate untrusted data, use meaningful internal types, and make exception handling and cleanup ownership explicit.
- **Comments:** Document contracts and intent, and preserve numbered input, processing, and return steps.
- **Safe changes:** Distinguish behavior-preserving refactoring from bug fixes and API changes. Run supported project commands in order: format, lint, then tests. Fix failures and restart from formatting until verification passes.

It supports three modes: review without editing, authorized refactoring, and teaching with original TypeScript examples. Without an explicit target, reviews cover current staged, unstaged, and relevant untracked changes. A mandatory applicability checklist tracks requirements and is rechecked after repairs. It is not a general security audit or a formatting-only tool.

Read the [skill instructions](skills/code-complete-typescript/SKILL.md) for its workflow and links to detailed references.

Numbered step comments use `1.` for input, `2.` for processing, and `3.` for return. Write the useful description directly after the number, without category labels:

```ts
// 1. Normalize the supplied text.
// 2. Collect unique tokens in their original order.
// 3. Return the token count.
```

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

After installation, ask your agent to use the skill for a bounded task. For example:

```text
Use code-complete-typescript to review the current TypeScript changes.
Report readability and maintainability findings without editing files.
```

For an authorized refactor:

```text
Use code-complete-typescript to improve the readability of this module.
Preserve behavior, keep the changes focused, and run the relevant checks.
```

The skill distinguishes review, refactoring, and teaching. It requires explicit attention to observable behavior, failure paths, ownership, and verification.

## Layout

```text
skills/
└── code-complete-typescript/
    ├── SKILL.md          # Entry point and workflow.
    ├── agents/           # Agent UI metadata.
    └── references/       # Criteria, research, and TypeScript examples.
```

Start with a skill's `SKILL.md`; open its supporting references as needed.
