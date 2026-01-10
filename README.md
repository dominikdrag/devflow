# Devflow Plugin

Comprehensive feature development workflows with specialized agents for codebase exploration, architecture design, test planning, security auditing, and quality review.

## Overview

This plugin provides two development workflow commands:
- **`/feature`** - 9-phase implementation-first workflow
- **`/tdd`** - 9-phase Test-Driven Development workflow with Red-Green-Refactor cycles

Both workflows ensure deep codebase understanding before implementation and use specialized agents powered by different models for optimal results.

## Agents

| Agent | Model | Color | Purpose | Used By |
|-------|-------|-------|---------|---------|
| `code-explorer` | Sonnet | Yellow | Analyzes codebase, traces execution paths, maps architecture | Both |
| `code-architect` | **Opus** | Yellow | Designs feature architectures with comprehensive blueprints | Both |
| `test-analyzer` | **Opus** | Green | Analyzes existing code and proposes test plans | `/feature` |
| `tdd-test-planner` | **Opus** | Green | Designs tests from requirements before code exists | `/tdd` |
| `test-runner` | Haiku | Green | Executes tests and reports structured results | Both |
| `code-reviewer` | **Opus** | Red | Reviews code for bugs, security, and convention adherence | Both |
| `security-auditor` | Sonnet | Red | Deep security analysis (optional, on request) | Both |

### Agent Triggering

Agents include enhanced descriptions for automatic triggering:

- **code-explorer**: Triggers when exploring unfamiliar code or tracing existing features
- **code-architect**: Triggers when designing new features or planning implementations
- **test-analyzer**: Triggers when analyzing existing code to propose test cases
- **tdd-test-planner**: Triggers when designing tests from requirements (TDD workflow)
- **test-runner**: Triggers when executing tests and reporting results
- **code-reviewer**: Triggers after code changes or on review requests
- **security-auditor**: Triggers only on explicit user request (optional)

## Commands

### `/feature [options] <feature-description>`

Launches the guided 9-phase **implementation-first** workflow:

1. **Discovery** - Understand requirements
2. **Codebase Exploration** - Learn existing patterns with parallel explorer agents
3. **Clarifying Questions** - Iterative dialogue to resolve all ambiguities
4. **Architecture Design** - Design approach with architect agents (user selects from options)
5. **Planning** - Create implementation plan with task-level tracking (user approval required)
6. **Implementation** - Build following approved plan, update task progress in plan file
7. **Testing** - Reconcile test-analyzer proposals with planned TEST-NNN tasks (user approval required), update plan, write tests, run with test-runner
8. **Quality Review** - Review with parallel reviewer agents, reconcile findings with REVIEW-NNN tasks, present for user selection, update plan (+ optional security audit)
9. **Summary** - Document completion, clean up state file (plan file kept if incomplete)

### `/tdd [options] <feature-description>`

Launches the guided 9-phase **test-first** TDD workflow:

1. **Discovery** - Understand requirements
2. **Codebase Exploration** - Learn existing patterns with parallel explorer agents
3. **Clarifying Questions** - Iterative dialogue to resolve all ambiguities
4. **Test Planning** - Design tests from requirements with `tdd-test-planner` (user approval required)
5. **Architecture Design** - Design code to pass planned tests (user selects from options)
6. **Planning** - Create TDD task list with Red/Green/Refactor substeps (user approval required)
7. **TDD Implementation** - Per-task Red-Green-Refactor cycles:
   - **RED**: Write failing test, verify it fails
   - **GREEN**: Write minimal code to pass (max 3 retries)
   - **REFACTOR**: Optional cleanup while keeping tests green
8. **Quality Review** - Review with parallel reviewer agents (+ optional security audit)
9. **Summary** - Document completion, report test coverage

**Key Difference**: In TDD, tests are designed BEFORE architecture, and implementation is interleaved with testing per-task.

## Workflow Enforcement

The plugin includes gates that enforce the development workflow:

- **Exploration Completion Gate**: Phase 3 (Clarifying Questions) cannot begin until ALL exploration agents have returned their complete output
- **Architecture Selection Gate**: Planning phase cannot begin until user explicitly selects an architecture via `AskUserQuestion` (or provides their own approach)
- **Planning Approval Gate**: Implementation cannot begin until user explicitly approves the implementation plan
- **Quality Review Gate**: Fixes are not applied until user reviews consolidated findings and explicitly selects which issues to address

### Phase Scope Rules

#### `/feature` Workflow
Each phase works exclusively on its designated task type:
- **Phase 6 (Implementation)**: Works ONLY on `TASK-NNN` tasks
- **Phase 7 (Testing)**: Works ONLY on `TEST-NNN` tasks
- **Phase 8 (Review)**: Works ONLY on `REVIEW-NNN` tasks

#### `/tdd` Workflow
TDD tasks use a different structure with Red-Green-Refactor substeps:
- **Phase 7 (TDD Implementation)**: Works on `TDD-NNN` tasks with substeps:
  - `TDD-NNN-RED`: Write failing tests
  - `TDD-NNN-GREEN`: Implement to pass
  - `TDD-NNN-REFACTOR`: Optional cleanup
- **Phase 8 (Review)**: Works on `REVIEW-NNN` tasks (same as /feature)

This separation ensures clear tracking and prevents scope creep between phases.

## Configuration

### Agent Count Flags

Control the number of agents launched per phase:

#### `/feature` Flags

| Flag | Default | Range | Phase |
|------|---------|-------|-------|
| `--explorers=N` | 3 | 1-10 | Phase 2: Codebase Exploration |
| `--architects=N` | 3 | 1-5 | Phase 4: Architecture Design |
| `--analyzers=N` | 1 | 1-5 | Phase 7: Testing |
| `--reviewers=N` | 3 | 1-5 | Phase 8: Quality Review |

#### `/tdd` Flags

| Flag | Default | Range | Phase |
|------|---------|-------|-------|
| `--explorers=N` | 3 | 1-10 | Phase 2: Codebase Exploration |
| `--planners=N` | 1 | 1-3 | Phase 4: Test Planning |
| `--architects=N` | 3 | 1-5 | Phase 5: Architecture Design |
| `--reviewers=N` | 3 | 1-5 | Phase 8: Quality Review |

### Examples

```bash
# /feature examples
/feature --explorers=1 --architects=1 Add a utility function
/feature --reviewers=5 Implement payment processing

# /tdd examples
/tdd Add user authentication with OAuth support
/tdd --planners=2 --architects=2 Implement complex business logic
/tdd --explorers=1 --architects=1 --reviewers=1 Small TDD feature
```

## Usage

```bash
# /feature - implementation first
/feature Add user authentication with OAuth support
/feature  # interactive mode

# /tdd - test first
/tdd Add user authentication with OAuth support
/tdd  # interactive mode
```

## When to Use

### Use `/feature` for:
- New features touching multiple files
- Features requiring architectural decisions
- Complex integrations with existing code
- Features with unclear requirements

### Use `/tdd` for:
- Features where you want tests to drive design decisions
- Complex business logic that benefits from test-first thinking
- Features with well-defined acceptance criteria
- When you want comprehensive test coverage as a natural byproduct

### Skip both for:
- Single-line bug fixes
- Trivial, obvious changes
- Urgent hotfixes
- Exploratory prototyping

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
