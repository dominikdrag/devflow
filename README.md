# Feature Development Plugin

Comprehensive feature development workflow with specialized agents for codebase exploration, architecture design, testing, security auditing, and quality review.

## Overview

This plugin guides you through a systematic 8-phase feature development process that ensures deep codebase understanding before implementation. It uses specialized agents powered by different models for optimal results.

## Agents

| Agent | Model | Color | Purpose |
|-------|-------|-------|---------|
| `code-explorer` | Sonnet | Yellow | Analyzes codebase, traces execution paths, maps architecture |
| `code-architect` | **Opus** | Green | Designs feature architectures with comprehensive blueprints |
| `test-writer` | **Opus** | Green | Writes comprehensive tests after implementation |
| `security-auditor` | Sonnet | Red | Deep security analysis (optional, on request) |
| `code-reviewer` | **Opus** | Red | Reviews code for bugs, security, and convention adherence |

### Agent Triggering

Agents include enhanced descriptions for automatic triggering:

- **code-explorer**: Triggers when exploring unfamiliar code or tracing existing features
- **code-architect**: Triggers when designing new features or planning implementations
- **test-writer**: Triggers after implementation or when tests are requested
- **security-auditor**: Triggers only on explicit user request (optional)
- **code-reviewer**: Triggers after code changes or on review requests

## Command

### `/feature-dev [feature-description]`

Launches the guided 8-phase workflow:

1. **Discovery** - Understand requirements
2. **Codebase Exploration** - Learn existing patterns with parallel explorer agents
3. **Clarifying Questions** - Resolve all ambiguities
4. **Architecture Design** - Design approach with architect agents (requires approval)
5. **Implementation** - Build following approved architecture
6. **Testing** - Write comprehensive tests with test-writer agent
7. **Quality Review** - Review with parallel reviewer agents (+ optional security audit)
8. **Summary** - Document completion

## Workflow Enforcement

The plugin includes hooks that enforce the development workflow:

- **Architecture Approval Gate**: Implementation cannot begin until architecture is explicitly approved by the user

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

## Security Features

The optional `security-auditor` agent performs deep analysis including:
- OWASP Top 10 vulnerability checks
- Authentication and authorization review
- Input validation analysis
- Sensitive data handling verification

To request a security audit, ask during the Quality Review phase.

## Credits

Based on Anthropic's official feature-dev plugin, enhanced with:
- Opus models for architecture, testing, and code review
- Comprehensive testing phase with test-writer agent
- Optional security auditing with security-auditor agent
- Workflow enforcement hooks
- Enhanced agent descriptions for better auto-triggering
