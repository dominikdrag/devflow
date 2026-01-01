# Feature Development Plugin

Comprehensive feature development workflow with specialized agents for codebase exploration, architecture design, security auditing, and quality review.

## Overview

This plugin guides you through a systematic 8-phase feature development process that ensures deep codebase understanding before implementation. It uses specialized agents powered by different models for optimal results.

## Agents

| Agent | Model | Color | Purpose |
|-------|-------|-------|---------|
| `code-explorer` | Sonnet | Yellow | Analyzes codebase, traces execution paths, maps architecture |
| `code-architect` | **Opus** | Green | Designs feature architectures with comprehensive blueprints |
| `test-analyzer` | Sonnet | Yellow | Analyzes changes and proposes comprehensive test plans |
| `security-auditor` | Sonnet | Red | Deep security analysis (optional, on request) |
| `code-reviewer` | **Opus** | Red | Reviews code for bugs, security, and convention adherence |

### Agent Triggering

Agents include enhanced descriptions for automatic triggering:

- **code-explorer**: Triggers when exploring unfamiliar code or tracing existing features
- **code-architect**: Triggers when designing new features or planning implementations
- **test-analyzer**: Triggers when analyzing code to propose test cases
- **security-auditor**: Triggers only on explicit user request (optional)
- **code-reviewer**: Triggers after code changes or on review requests

## Command

### `/feature-dev [options] <feature-description>`

Launches the guided 8-phase workflow:

1. **Discovery** - Understand requirements
2. **Codebase Exploration** - Learn existing patterns with parallel explorer agents
3. **Clarifying Questions** - Resolve all ambiguities
4. **Architecture Design** - Design approach with architect agents (user selects from options)
5. **Implementation** - Build following approved architecture
6. **Testing** - Analyze with test-analyzer, then write tests directly (preserves context)
7. **Quality Review** - Review with parallel reviewer agents (+ optional security audit)
8. **Summary** - Document completion

## Workflow Enforcement

The plugin includes hooks that enforce the development workflow:

- **Architecture Selection Gate**: Implementation cannot begin until user explicitly selects an architecture via `AskUserQuestion` (or provides their own approach)

## Configuration

### Agent Count Flags

Control the number of agents launched per phase:

| Flag | Default | Range | Phase |
|------|---------|-------|-------|
| `--explorers=N` | 3 | 1-5 | Phase 2: Codebase Exploration |
| `--architects=N` | 3 | 1-5 | Phase 4: Architecture Design |
| `--analyzers=N` | 1 | 1-5 | Phase 6: Testing |
| `--reviewers=N` | 3 | 1-5 | Phase 7: Quality Review |

### Examples

```bash
# Use fewer agents for smaller features
/feature-dev --explorers=1 --architects=1 Add a utility function

# More reviewers for critical features
/feature-dev --reviewers=5 Implement payment processing

# Minimal agents for quick iteration
/feature-dev --explorers=1 --architects=1 --reviewers=1 Small change
```

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

Available via [dominikdrag-marketplace](https://github.com/dominikdrag/dominikdrag-marketplace). Run from Claude Code CLI:

```
/plugin marketplace add dominikdrag/dominikdrag-marketplace
/plugin install feature-dev@dominikdrag-marketplace
```

## Security Features

The optional `security-auditor` agent performs deep analysis including:
- OWASP Top 10 vulnerability checks
- Authentication and authorization review
- Input validation analysis
- Sensitive data handling verification

To request a security audit, ask during the Quality Review phase.

## Acknowledgements

This plugin is based on [Anthropic's official feature-dev plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev).

**Enhancements over the original:**
- Opus models for architecture and code review
- Testing phase with test-analyzer for proposals, direct writing for context preservation
- Optional security auditing with security-auditor agent
- Workflow enforcement hooks
- Enhanced agent descriptions for better auto-triggering
