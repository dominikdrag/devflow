# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Claude Code plugin that provides a 9-phase feature development workflow. It uses specialized agents for codebase exploration, architecture design, test analysis, security auditing, and code review.

## Plugin Structure

```
devflow/
├── .claude-plugin/plugin.json    # Plugin manifest (name, version, description)
├── agents/                        # Agent definitions (markdown with YAML frontmatter)
│   ├── code-explorer.md          # Sonnet - traces execution paths, maps architecture
│   ├── code-architect.md         # Opus - designs feature architectures
│   ├── code-reviewer.md          # Opus - reviews for bugs and convention adherence
│   ├── test-analyzer.md          # Sonnet - analyzes code, proposes test cases
│   └── security-auditor.md       # Sonnet - OWASP Top 10 checks (optional)
├── commands/feature.md           # Main workflow command definition
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

The plugin uses a `PreToolUse` hook on `Write|Edit` operations to enforce the architecture approval gate. The prompt-based hook checks that the user has explicitly selected an architecture via `AskUserQuestion` before allowing implementation.

If the prompt-based approach proves unreliable, NOTES.md documents an alternative state-file approach using `claude-tmp/devflow-approved.tmp`.

## Workflow Phases

1. Discovery - Understand requirements
2. Codebase Exploration - Launch 3 parallel `code-explorer` agents
3. Clarifying Questions - Resolve ambiguities
4. Architecture Design - Launch 3 `code-architect` agents, present options via `AskUserQuestion`
5. Planning - Create implementation plan with task-level tracking (`claude-tmp/devflow-plan.md`)
6. Implementation - Build following approved plan, update task progress
7. Testing - Launch `test-analyzer` agent for proposals, then write tests directly (preserves context)
8. Quality Review - Launch 3 parallel `code-reviewer` agents (+ optional `security-auditor`)
9. Summary - Document completion, clean up state and plan files

## Key Design Decisions

- Opus models used for architecture and code review (higher reasoning quality)
- Sonnet used for exploration, test analysis, and security auditing (cost efficiency for analysis tasks)
- Tests written directly (not by subagent) to preserve implementation context
- Confidence thresholds: code-reviewer >= 80/100, security-auditor >= 85/100
- Architecture selection is a hard gate - implementation blocked until user explicitly chooses

## Commit Convention

Use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/):

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`

**Breaking changes:** Append `!` after type/scope (e.g., `feat!:`) or add `BREAKING CHANGE:` footer
