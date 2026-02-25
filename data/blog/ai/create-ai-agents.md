---
title: 'Building AI Agents with Claude Code and Cursor'
date: 2026-02-24
lastmod: '2026-02-24'
tags: ['engineering', 'ai', 'llm']
draft: false
summary: 'Learn how to build custom AI agents using Claude Code and Cursor — from defining system instructions to creating autonomous code-review workflows that enforce your project conventions'
layout: PostSimple
authors: ['akash-srivastava']
---

## Introduction

AI-powered coding assistants have evolved far beyond simple autocomplete. Today, tools like **Claude Code** and **Cursor** allow developers to build **custom agents** — autonomous workflows that can read your codebase, make decisions, and execute multi-step tasks with minimal human intervention.

Whether you want an agent that reviews pull requests, runs migrations, generates documentation, or enforces coding standards — you can build it. This guide walks you through the fundamentals of agent creation, the differences between platforms, and practical examples you can adapt to your own projects.

---

## What is an AI Agent?

An AI agent is more than a chatbot. While a chatbot responds to a single prompt and stops, an agent:

- **Plans** a sequence of steps to accomplish a goal
- **Uses tools** (file reads, terminal commands, web searches) to gather context
- **Makes decisions** based on intermediate results
- **Iterates** until the task is complete or it needs human input

Think of it as the difference between asking someone a question and hiring someone to do a job. A question gets you an answer. An agent gets you a result.

### Key Properties of a Good Agent

| Property          | Description                                                               |
| ----------------- | ------------------------------------------------------------------------- |
| **Autonomous**    | Can complete multi-step tasks without constant prompting                  |
| **Context-Aware** | Reads and understands your codebase, conventions, and history             |
| **Tool-Using**    | Leverages file I/O, terminal commands, APIs, and browser tools            |
| **Iterative**     | Adjusts its approach based on feedback (test failures, lint errors, etc.) |
| **Scoped**        | Knows its boundaries — what it should and shouldn't do                    |

---

## The Two Approaches: Claude Code vs. Cursor

Both Claude Code and Cursor support agent-like workflows, but they approach it differently.

### Claude Code (CLI-Based Agents)

Claude Code is Anthropic's official CLI tool. It runs in your terminal and has deep access to your development environment — files, git, shell commands, and more.

**How agents work in Claude Code:**

- You define agent behavior through **system prompts**, **CLAUDE.md project instructions**, and **custom slash commands** (skills)
- Agents are invoked via the **Task tool** (subagents) or through **custom hooks**
- Agents can run in the **foreground** (blocking) or **background** (parallel)
- You can configure agents with specific **tool permissions** and **isolation modes** (e.g., git worktrees)

**Key files for agent configuration:**

```
project-root/
├── CLAUDE.md                    # Project-level instructions (always loaded)
├── .claude/
│   ├── settings.json            # Permission settings, MCP servers
│   ├── commands/                # Custom slash commands (skills)
│   │   ├── review.md            # /review command
│   │   └── deploy-check.md     # /deploy-check command
│   └── hooks/                   # Event-driven automation
```

### Cursor (IDE-Based Agents)

Cursor is a VS Code fork with built-in AI capabilities. Its agent mode runs directly inside the editor.

**How agents work in Cursor:**

- You define behavior through **`.cursorrules`** files (project-level instructions)
- Agents operate in **Agent Mode** (Cmd+I or the composer panel)
- Cursor agents can read files, edit code, run terminal commands, and search the web
- You guide agents with **rules files** that act as persistent system prompts

**Key files for agent configuration:**

```
project-root/
├── .cursorrules                 # Project-level instructions (always loaded)
├── .cursor/
│   └── rules/                   # Additional rule files
│       ├── code-review.mdc      # Code review rules
│       └── testing.mdc          # Testing rules
```

---

## Core Concepts for Building Agents

Regardless of which tool you use, effective agents share common building blocks.

### 1. System Instructions (The Agent's Personality)

System instructions define _who_ the agent is and _how_ it behaves. They are the most important piece of any agent.

**What to include in system instructions:**

- **Role definition**: What is this agent responsible for?
- **Conventions**: What patterns, naming rules, and standards should it follow?
- **Boundaries**: What should it NOT do?
- **Output format**: How should it report findings?
- **Tool preferences**: Which tools should it use and how?

> **Tip**: The more specific your instructions, the more reliable your agent. Vague instructions like "review the code" produce vague results. Specific instructions like "check for missing TypeScript types, ensure no `any` usage, verify test coverage for new functions" produce actionable output.

### 2. Context Injection (What the Agent Knows)

Agents are only as good as the context they receive. There are several ways to inject context:

- **Project instruction files** (`CLAUDE.md`, `.cursorrules`) — loaded automatically on every interaction
- **File reads** — the agent reads specific files during execution
- **Codebase search** — the agent searches for patterns, definitions, or usages
- **Git context** — diffs, commit history, branch comparisons
- **External data** — API docs, web searches, MCP servers

### 3. Tool Access (What the Agent Can Do)

Agents need tools to be effective. Common tool categories:

| Tool Category       | Examples                                     |
| ------------------- | -------------------------------------------- |
| **File Operations** | Read, write, edit, search files              |
| **Terminal**        | Run build commands, tests, linters           |
| **Git**             | Check diffs, blame, history, create commits  |
| **Web**             | Fetch documentation, search for solutions    |
| **Browser**         | Interact with running applications (via MCP) |

### 4. Iteration Loops (How the Agent Improves)

The best agents don't just execute — they verify and iterate:

```
Plan → Execute → Verify → Fix → Verify again
```

For example, a code-review agent might:

1. Read the changed files
2. Check for convention violations
3. Run the linter
4. Run the type checker
5. Report findings
6. (Optionally) auto-fix issues and re-verify

---

## Example: Building a Code-Review Agent

Let's build a practical code-review agent that enforces project conventions. We'll show implementations for both Claude Code and Cursor.

### What Our Agent Will Do

1. Identify changed files (from git diff)
2. Check for TypeScript type safety (no `any` usage)
3. Verify naming conventions (CamelCase files, camelCase variables)
4. Ensure test files exist for new components
5. Run the linter and type checker
6. Produce a structured report

---

### Claude Code Implementation

#### Step 1: Define Project Instructions (`CLAUDE.md`)

Your `CLAUDE.md` file sets the baseline conventions that ALL agents (including the code-review agent) will follow:

```markdown
# Project Conventions

## Naming Conventions

- **Files/Folders**: Use CamelCase
- **Variables**: Use camelCase format
- **Enums/Constants**: Use CAPITAL_LETTERS_WITH_UNDERSCORES
- **Interfaces**: Required instead of `any` type

## Quality Gates

- TypeScript compilation passes (`npx tsc --noEmit`)
- ESLint violations resolved
- Unit tests written for non-trivial changes
- All tests passing
- No `any` types — use proper interfaces
```

#### Step 2: Create a Custom Slash Command

Create a file at `.claude/commands/review.md`:

```markdown
# Code Review Agent

Review the current branch against main and check for violations
of our project conventions.

## Steps

1. Run `git diff main...HEAD --name-only` to get all changed files
2. For each changed file:
   - Read the file contents
   - Check for `any` type usage — flag every instance
   - Verify file naming follows CamelCase convention
   - Check that new components in `/src/` have a corresponding `.test.tsx` file
3. Run `npx tsc --noEmit` and capture any type errors
4. Run `npx eslint` on all changed files and capture violations
5. Produce a structured report in this format:

## Review Report

### Type Safety

- [ ] No `any` usage found
- List all violations with file:line references

### Naming Conventions

- [ ] All files follow CamelCase
- List all violations

### Test Coverage

- [ ] All new components have test files
- List any missing test files

### Linting

- [ ] ESLint passes with no errors
- List all errors (warnings are acceptable)

### TypeScript Compilation

- [ ] `tsc --noEmit` passes
- List all type errors

### Summary

- Total issues: X
- Blocking issues: X
- Recommendation: APPROVE / REQUEST CHANGES
```

#### Step 3: Run It

```bash
claude
> /review
```

Claude Code will execute the steps autonomously — reading files, running commands, and producing the structured report.

#### Advanced: Using Subagents (Task Tool)

For larger codebases, you can define the review agent as a **subagent type** in your Claude Code configuration. This allows it to run in parallel with other tasks:

```markdown
<!-- In your CLAUDE.md or agent configuration -->

- code-review: Use this agent when code has been recently written or
  modified. It reviews changes against CLAUDE.md conventions, checks
  TypeScript types, naming conventions, test coverage, and linting.
  Invoke proactively after completing features or bug fixes.
```

When invoked as a subagent, the code-review agent runs in its own context window, preventing large diffs from consuming the main conversation's context.

---

### Cursor Implementation

#### Step 1: Create a Rules File

Create `.cursor/rules/code-review.mdc`:

```markdown
---
description: Code review rules for enforcing project conventions
globs: ['src/**/*.ts', 'src/**/*.tsx']
alwaysApply: false
---

# Code Review Agent

When asked to review code, follow these steps:

## Conventions to Check

### Type Safety

- Flag any usage of the `any` type
- Ensure all function parameters have explicit types
- Ensure all function return types are declared
- Interfaces should be used for object shapes

### Naming

- File names: CamelCase (e.g., `UserProfile.tsx`, `ApiService.ts`)
- Variables: camelCase (e.g., `userName`, `isActive`)
- Constants/Enums: UPPER_SNAKE_CASE (e.g., `MAX_RETRIES`, `API_URL`)

### Test Coverage

- Every new component must have a `.test.tsx` file in the same directory
- Tests should use React Testing Library
- Test names should describe behavior, not implementation

### Architecture

- No direct imports from cloud service packages outside `/src/services/`
- Use abstraction hooks: `useAuth()`, `useFeatureFlags()`, `useAnalytics()`
- Prefer function components over class components

## Output Format

Present findings as a checklist:

- [ ] or [x] for each category
- File:line references for each violation
- Severity: 🔴 Blocking / 🟡 Warning / 🟢 Pass
```

#### Step 2: Use Agent Mode

Open Cursor, press `Cmd+I` (or `Ctrl+I`), and type:

```
Review all files changed on this branch compared to main.
Use the code-review rules.
```

Cursor's agent will read your `.mdc` rules, scan the changed files, and produce a review report.

---

## Building Other Types of Agents

The code-review agent is just one example. Here are other agents you can build using the same patterns:

### Documentation Agent

```markdown
# Documentation Agent

When new public functions or components are added:

1. Check if JSDoc comments exist
2. Verify README sections are up to date
3. Ensure API changes are reflected in docs/
4. Generate missing documentation stubs
```

### Migration Agent

```markdown
# Migration Agent

When database schema changes are detected:

1. Generate a migration file with up/down methods
2. Verify the migration is reversible
3. Check for data loss risks
4. Run the migration against a test database
5. Verify all existing tests still pass
```

### Security Audit Agent

```markdown
# Security Audit Agent

Scan changed files for:

1. Hardcoded secrets, API keys, or tokens
2. SQL injection vulnerabilities (unsanitized inputs)
3. XSS risks (unescaped user content in JSX)
4. Insecure dependencies (run `npm audit`)
5. Exposed environment variables in client-side code
```

### PR Summary Agent

```markdown
# PR Summary Agent

1. Read all commits on the current branch vs main
2. Read the full diff
3. Generate:
   - A concise PR title (under 70 characters)
   - A bulleted summary of changes
   - A test plan checklist
   - A list of files that reviewers should focus on
```

---

## Best Practices for Agent Design

### 1. Be Explicit, Not Implicit

```
❌ "Review the code for issues"
✅ "Check all .tsx files changed on this branch for: any type usage,
    missing test files, CamelCase file naming, and direct cloud
    service imports outside /src/services/"
```

### 2. Define Output Format

Agents produce better results when they know exactly what format to output. Specify:

- Checklists vs. prose
- File:line reference format
- Severity levels
- Summary structure

### 3. Include Verification Steps

Always include a "verify your work" step:

```markdown
After making changes:

1. Run `npx tsc --noEmit` — fix any errors
2. Run `npx eslint [changed files]` — fix violations
3. Run `npm test -- [related tests]` — ensure tests pass
4. Only report completion after all checks pass
```

### 4. Set Boundaries

Tell the agent what NOT to do:

```markdown
## Boundaries

- Do NOT modify files — only report findings
- Do NOT create new files
- Do NOT run destructive commands (rm, git reset, etc.)
- Do NOT push to remote repositories
- Ask the user before making any changes
```

### 5. Use Iteration Loops

The most effective agents follow a loop:

```
Analyze → Act → Verify → Repeat if needed
```

Build this loop directly into your agent instructions:

```markdown
## Workflow

1. Identify all issues
2. Fix the first issue
3. Re-run verification (linter, type checker, tests)
4. If new issues appear, fix them
5. Repeat until all checks pass
6. Report the final state
```

### 6. Scope Your Agents Narrowly

One focused agent is better than one that tries to do everything:

```
❌ One agent: "Review code, fix bugs, write tests, update docs, and deploy"
✅ Four agents: review, test-writer, doc-generator, deploy-checker
```

Small, focused agents are easier to debug, test, and improve.

---

## Claude Code vs. Cursor: When to Use Which

| Scenario                      | Recommended Tool                          |
| ----------------------------- | ----------------------------------------- |
| CI/CD pipeline integration    | Claude Code (CLI-native)                  |
| Interactive development       | Cursor (IDE-native)                       |
| Background automation         | Claude Code (background agents)           |
| Visual code review            | Cursor (inline editor)                    |
| Multi-repo workflows          | Claude Code (flexible working dirs)       |
| Team onboarding               | Either (both support project-level rules) |
| Complex multi-step automation | Claude Code (subagent orchestration)      |

---

## Getting Started Checklist

Here's how to build your first agent today:

1. **Choose your tool**: Claude Code (CLI) or Cursor (IDE)
2. **Document your conventions**: Create a `CLAUDE.md` or `.cursorrules` file
3. **Pick a single task**: Start small — a linter, a naming checker, a test verifier
4. **Write explicit instructions**: Define steps, output format, and boundaries
5. **Test it on real code**: Run the agent against your current branch
6. **Iterate on the instructions**: Refine based on what the agent gets wrong
7. **Share with your team**: Commit the configuration files to version control

---

## Conclusion

AI agents are not magic — they are **automated workflows powered by clear instructions**. The quality of your agent is directly proportional to the quality of your instructions.

Start with a simple, focused agent like the code-review example above. Once you see how it works, expand to other workflows: testing, documentation, security, deployment checks. The patterns are the same — only the instructions change.

The most important thing is to **start**. Write a `CLAUDE.md` or `.cursorrules` file today. Define one convention. Build one agent. Iterate from there.

---
