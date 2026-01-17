# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Claude Code plugin that provides two development workflow commands:
- `/feature` - 9-phase feature development workflow (implementation-first)
- `/tdd` - 9-phase TDD workflow with Red-Green-Refactor cycles (test-first)

Both use specialized agents for codebase exploration, architecture design, test planning, security auditing, and code review.

## Plugin Structure

```
devflow/
├── .claude-plugin/plugin.json    # Plugin manifest (name, version, description)
├── agents/                        # Agent definitions (markdown with YAML frontmatter)
│   ├── code-explorer.md          # Sonnet - traces execution paths, maps architecture
│   ├── code-architect.md         # Opus - designs feature architectures
│   ├── code-reviewer.md          # Opus - reviews for bugs and convention adherence
│   ├── test-analyzer.md          # Opus - analyzes code, proposes test cases
│   ├── test-runner.md            # Haiku - executes tests and reports results
│   ├── tdd-test-planner.md       # Opus - designs tests from requirements (TDD)
│   └── security-auditor.md       # Opus - OWASP Top 10 checks (optional)
├── commands/
│   ├── feature.md                # Standard workflow command
│   └── tdd.md                    # TDD workflow command
└── hooks/hooks.json              # Workflow enforcement hooks
```

## Agent Configuration

Agents are defined in markdown files with YAML frontmatter:

```yaml
---
name: agent-name
description: When-to-use description for auto-triggering
tools: Glob, Grep, LS, Read, ...  # Allowed tools
model: sonnet|opus                 # Model to use
color: yellow|green|red           # Status line color
---
[System prompt content]
```

## Hook System

The plugin uses a `PreCompact` hook to persist workflow state before conversation compaction. The hook saves state to the appropriate state file (`devflow-feature-state.json` for `/feature`, `devflow-tdd-state.json` for `/tdd`) enabling workflow resumption.

## Workflow Phases

### /feature Workflow (Implementation-First)

1. Discovery - Understand requirements
2. Codebase Exploration - Launch 3 parallel `code-explorer` agents
3. Clarifying Questions - Resolve ambiguities
4. Architecture Design - Launch 3 `code-architect` agents, present options via `AskUserQuestion`
5. Planning - Create implementation plan with task-level tracking (`claude-tmp/devflow-plan.md`)
6. Implementation - Build following approved plan, update task progress
7. Testing - Launch `test-analyzer` agent for proposals, then write tests directly (preserves context)
8. Quality Review - Launch 3 parallel `code-reviewer` agents (+ optional `security-auditor`)
9. Summary - Document completion, clean up state and plan files

### /tdd Workflow (Test-First)

1. Discovery - Understand requirements
2. Codebase Exploration - Launch 3 parallel `code-explorer` agents
3. Clarifying Questions - Resolve ambiguities
4. **Test Planning** - Launch `tdd-test-planner` agent to design tests from requirements
5. Architecture Design - Launch 3 `code-architect` agents, design code to pass planned tests
6. Planning - Create TDD task list with Red/Green/Refactor substeps (`claude-tmp/tdd-plan.md`)
7. **TDD Implementation** - Per-task Red-Green-Refactor cycles:
   - RED: Write failing test, verify it fails
   - GREEN: Write minimal code to pass (max 3 retries)
   - REFACTOR: Optional cleanup while keeping tests green
8. Quality Review - Launch 3 parallel `code-reviewer` agents (end of workflow)
9. Summary - Document completion, report test coverage

**Key Difference**: In TDD, tests are designed BEFORE architecture (Phase 4 before Phase 5), and implementation is interleaved with testing per-task.

## Key Design Decisions

- Opus models used for architecture, code review, test planning, and security auditing (higher reasoning quality for nuanced judgment)
- Sonnet used for exploration (fast information gathering; synthesis done by main model)
- Haiku used for test execution (fast, simple task)
- Tests written directly (not by subagent) to preserve implementation context
- Confidence thresholds: code-reviewer >= 80/100, security-auditor >= 85/100
- Architecture selection is a hard gate - implementation blocked until user explicitly chooses
- TDD workflow uses separate state files (`devflow-tdd-state.json`, `tdd-plan.md`) from feature workflow

## Commit Convention

Use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/):

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`

**Breaking changes:** Append `!` after type/scope (e.g., `feat!:`) or add `BREAKING CHANGE:` footer
