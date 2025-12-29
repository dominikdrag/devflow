# Feature Development Plugin

Comprehensive feature development workflow with specialized agents for codebase exploration, architecture design, and quality review.

## Overview

This plugin guides you through a systematic 7-phase feature development process that ensures deep codebase understanding before implementation. It uses specialized agents powered by different models for optimal results.

## Agents

| Agent | Model | Purpose |
|-------|-------|---------|
| `code-explorer` | Sonnet | Analyzes codebase, traces execution paths, maps architecture |
| `code-architect` | **Opus** | Designs feature architectures with comprehensive blueprints |
| `code-reviewer` | **Opus** | Reviews code for bugs, security, and convention adherence |

## Command

### `/feature-dev`

Launches the guided 7-phase workflow:

1. **Discovery** - Understand requirements
2. **Codebase Exploration** - Learn existing patterns with parallel explorer agents
3. **Clarifying Questions** - Resolve all ambiguities
4. **Architecture Design** - Design approach with architect agents
5. **Implementation** - Build following approved architecture
6. **Quality Review** - Review with parallel reviewer agents
7. **Summary** - Document completion

## Usage

```bash
# With feature description
/feature-dev Add user authentication with OAuth support

# Interactive mode
/feature-dev
```

## When to Use

**Ideal for:**
- New features touching multiple files
- Features requiring architectural decisions
- Complex integrations with existing code
- Features with unclear requirements

**Skip for:**
- Single-line bug fixes
- Trivial, obvious changes
- Urgent hotfixes

## Installation

Copy this plugin to your Claude Code plugins directory or use:

```bash
claude --plugin-dir /path/to/feature-dev
```

## Credits

Based on Anthropic's official feature-dev plugin, enhanced with Opus models for architecture and code review phases.
