# Devflow Plugin

Comprehensive feature development workflow with specialized agents for codebase exploration, architecture design, security auditing, and quality review.

## Overview

This plugin guides you through a systematic 9-phase feature development process that ensures deep codebase understanding before implementation. It uses specialized agents powered by different models for optimal results.

## Agents

| Agent | Model | Color | Purpose |
|-------|-------|-------|---------|
| `code-explorer` | Sonnet | Yellow | Analyzes codebase, traces execution paths, maps architecture |
| `code-architect` | **Opus** | Green | Designs feature architectures with comprehensive blueprints |
| `test-analyzer` | **Opus** | Yellow | Analyzes changes and proposes comprehensive test plans |
| `test-runner` | Haiku | Green | Executes tests and reports structured results |
| `security-auditor` | Sonnet | Red | Deep security analysis (optional, on request) |
| `code-reviewer` | **Opus** | Red | Reviews code for bugs, security, and convention adherence |

### Agent Triggering

Agents include enhanced descriptions for automatic triggering:

- **code-explorer**: Triggers when exploring unfamiliar code or tracing existing features
- **code-architect**: Triggers when designing new features or planning implementations
- **test-analyzer**: Triggers when analyzing code to propose test cases
- **test-runner**: Triggers when executing tests and reporting results
- **security-auditor**: Triggers only on explicit user request (optional)
- **code-reviewer**: Triggers after code changes or on review requests

## Command

### `/feature [options] <feature-description>`

Launches the guided 9-phase workflow:

1. **Discovery** - Understand requirements
2. **Codebase Exploration** - Learn existing patterns with parallel explorer agents
3. **Clarifying Questions** - Iterative dialogue to resolve all ambiguities
4. **Architecture Design** - Design approach with architect agents (user selects from options)
5. **Planning** - Create implementation plan with task-level tracking (user approval required)
6. **Implementation** - Build following approved plan, update task progress in plan file
7. **Testing** - Reconcile test-analyzer proposals with planned TEST-NNN tasks (user approval required), update plan, write tests, run with test-runner
8. **Quality Review** - Review with parallel reviewer agents, reconcile findings with REVIEW-NNN tasks, present for user selection, update plan (+ optional security audit)
9. **Summary** - Document completion, clean up state file (plan file kept if incomplete)

## Workflow Enforcement

The plugin includes gates that enforce the development workflow:

- **Exploration Completion Gate**: Phase 3 (Clarifying Questions) cannot begin until ALL exploration agents have returned their complete output
- **Architecture Selection Gate**: Planning phase cannot begin until user explicitly selects an architecture via `AskUserQuestion` (or provides their own approach)
- **Planning Approval Gate**: Implementation cannot begin until user explicitly approves the implementation plan
- **Quality Review Gate**: Fixes are not applied until user reviews consolidated findings and explicitly selects which issues to address

### Phase Scope Rules

Each phase works exclusively on its designated task type:
- **Phase 6 (Implementation)**: Works ONLY on `TASK-NNN` tasks
- **Phase 7 (Testing)**: Works ONLY on `TEST-NNN` tasks
- **Phase 8 (Review)**: Works ONLY on `REVIEW-NNN` tasks

This separation ensures clear tracking and prevents scope creep between phases.

## Configuration

### Agent Count Flags

Control the number of agents launched per phase:

| Flag | Default | Range | Phase |
|------|---------|-------|-------|
| `--explorers=N` | 3 | 1-10 | Phase 2: Codebase Exploration |
| `--architects=N` | 3 | 1-5 | Phase 4: Architecture Design |
| `--analyzers=N` | 1 | 1-5 | Phase 7: Testing |
| `--reviewers=N` | 3 | 1-5 | Phase 8: Quality Review |

### Examples

```bash
# Use fewer agents for smaller features
/feature --explorers=1 --architects=1 Add a utility function

# More reviewers for critical features
/feature --reviewers=5 Implement payment processing

# Minimal agents for quick iteration
/feature --explorers=1 --architects=1 --reviewers=1 Small change
```

## Usage

```bash
# With feature description
/feature Add user authentication with OAuth support

# Interactive mode
/feature
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
/plugin install devflow@dominikdrag-marketplace
```

## Security Features

The optional `security-auditor` agent performs deep analysis including:
- OWASP Top 10 vulnerability checks
- Authentication and authorization review
- Input validation analysis
- Sensitive data handling verification

To request a security audit, ask during the Quality Review phase.

## Acknowledgements

This plugin was originally based on [Anthropic's official feature-dev plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev) but has since diverged significantly.

**Enhancements over the original:**
- Opus models for architecture, test analysis, and code review
- Testing phase with test-analyzer for proposals (user approval required), direct writing for context preservation, test-runner for execution
- Test validation in implementation phase ensures existing tests pass before writing new ones
- Optional security auditing with security-auditor agent
- Workflow enforcement hooks
- Enhanced agent descriptions for better auto-triggering
