# Devflow Plugin

Comprehensive feature development workflows with specialized agents for codebase exploration, architecture design, test planning, security auditing, and quality review.

## Overview

This plugin provides three development workflow commands:
- **`/feature`** - 9-phase implementation-first workflow
- **`/tdd`** - 9-phase Test-Driven Development workflow with Red-Green-Refactor cycles
- **`/code-review`** - Standalone code review with parallel agents and confidence-based filtering

The full workflows ensure deep codebase understanding before implementation. All commands use specialized agents powered by different models for optimal results.

## Agents

| Agent | Model | Color | Purpose | Used By |
|-------|-------|-------|---------|---------|
| `code-explorer` | Sonnet | Yellow | Analyzes codebase, traces execution paths, maps architecture | `/feature`, `/tdd` |
| `code-architect` | **Opus** | Yellow | Designs feature architectures with comprehensive blueprints | `/feature`, `/tdd` |
| `test-analyzer` | **Opus** | Green | Analyzes existing code and proposes test plans | `/feature` |
| `tdd-test-planner` | **Opus** | Green | Designs tests from requirements before code exists | `/tdd` |
| `test-runner` | Haiku | Green | Executes tests and reports structured results | `/feature`, `/tdd` |
| `code-reviewer` | **Opus** | Red | Reviews code for bugs, security, and convention adherence | All |
| `security-auditor` | **Opus** | Red | Deep security analysis (optional, on request) | All |

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

### `/feature <feature-description>`

Launches the guided 9-phase **implementation-first** workflow:

1. **Discovery** - Understand requirements
2. **Codebase Exploration** - Select exploration focuses (interactive), launch parallel agents
3. **Clarifying Questions** - Iterative dialogue to resolve all ambiguities
4. **Architecture Design** - Select architecture perspectives (interactive), user selects from proposals
5. **Planning** - Create implementation plan with task-level tracking (user approval required)
6. **Implementation** - Build following approved plan, update task progress in plan file
7. **Testing** - Select test focuses (interactive), get user approval, create TEST-NNN tasks, write tests
8. **Quality Review** - Select review focuses (interactive), present findings for user selection
9. **Summary** - Document completion, clean up state file (plan file kept if incomplete)

### `/tdd <feature-description>`

Launches the guided 9-phase **test-first** TDD workflow:

1. **Discovery** - Understand requirements
2. **Codebase Exploration** - Select exploration focuses (interactive), launch parallel agents
3. **Clarifying Questions** - Iterative dialogue to resolve all ambiguities
4. **Test Planning** - Select test focuses (interactive), design tests from requirements
5. **Architecture Design** - Select architecture perspectives (interactive), design code to pass tests
6. **Planning** - Create TDD task list with Red/Green/Refactor substeps (user approval required)
7. **TDD Implementation** - Per-task Red-Green-Refactor cycles:
   - **RED**: Write failing test, verify it fails
   - **GREEN**: Write minimal code to pass (max 3 retries)
   - **REFACTOR**: Optional cleanup while keeping tests green
8. **Quality Review** - Select review focuses (interactive), present findings for user selection
9. **Summary** - Document completion, report test coverage

**Key Difference**: In TDD, tests are designed BEFORE architecture, and implementation is interleaved with testing per-task.

### `/code-review [options]`

Standalone code review using the same agents and focuses as Phase 8 of the full workflows.

**File targeting options:**
- `--staged` (default) - Review staged git changes
- `--unstaged` - Review unstaged working directory changes
- `--pr N` - Review changes in pull request #N
- `--commit HASH` - Review a specific commit
- Explicit file paths - Review specific files

**Flow:**
1. **Target Identification** - Parse arguments, determine files to review
2. **Focus Selection** - Choose from 6 review focuses (same as Phase 8)
3. **Launch Agents** - Parallel agents based on selected focuses
4. **Present Findings** - Full agent output, organized by severity
5. **Fix Selection** - Choose issues to fix: numbers, "critical", "all", or "skip"
6. **Apply Fixes** - Execute selected fixes
7. **Re-review Option** - Optionally re-run to verify fixes
8. **Summary** - Final stats and cleanup

## Workflow Enforcement

The plugin includes gates that enforce the development workflow:

- **Exploration Completion Gate**: Phase 3 (Clarifying Questions) cannot begin until ALL exploration agents have returned their complete output
- **Architecture Selection Gate**: Planning phase cannot begin until user explicitly types their architecture selection (or describes their own approach)
- **Planning Approval Gate**: Implementation cannot begin until user explicitly approves the implementation plan
- **Quality Review Gate**: Fixes are not applied until user reviews consolidated findings and explicitly selects which issues to address

### Phase Scope Rules

#### `/feature` Workflow
Each phase works exclusively on its designated task type:
- **Phase 6 (Implementation)**: Works ONLY on `TASK-NNN` tasks
- **Phase 7 (Testing)**: Creates `TEST-NNN` tasks from test-analyzer output, then executes them
- **Phase 8 (Review)**: Creates/refines `REVIEW-NNN` tasks from code-reviewer, then executes them

#### `/tdd` Workflow
TDD tasks use a different structure with Red-Green-Refactor substeps:
- **Phase 7 (TDD Implementation)**: Works on `TDD-NNN` tasks with substeps:
  - `TDD-NNN-RED`: Write failing tests
  - `TDD-NNN-GREEN`: Implement to pass
  - `TDD-NNN-REFACTOR`: Optional cleanup
- **Phase 8 (Review)**: Works on `REVIEW-NNN` tasks (same as /feature)

This separation ensures clear tracking and prevents scope creep between phases.

## Interactive Focus Selection

Both workflows prompt you to type your choices in key phases. You type which focuses to explore (e.g., "1,2,3" or "1-3"), and each selection launches a parallel specialized agent.

### Exploration Focuses (Phase 2)
| # | Focus | What it covers |
|---|-------|----------------|
| 1 | Similar Features | Existing features as templates, reusable patterns |
| 2 | Integration Points | Where feature connects to existing systems |
| 3 | Data Flow & State | How data moves, storage, state management |
| 4 | Entry Points | API surfaces, UI components, CLI commands |
| 5 | Patterns & Conventions | Design patterns, coding standards |
| 6 | Error Handling | Error paths, validation, failure modes |

### Architecture Perspectives (Phase 4/5)
| # | Perspective | Philosophy |
|---|-------------|------------|
| 1 | Minimal Changes | Smallest change, maximum reuse |
| 2 | Clean Architecture | Elegant abstractions, separation of concerns |
| 3 | Pragmatic Balance | Trade-off between speed and quality |
| 4 | Performance-First | Hot paths, latency-sensitive |
| 5 | Security-First | Auth, payments, PII |
| 6 | Scalability-First | >1000 concurrent users |
| 7 | Testability-First | Complex business logic |
| 8 | Migration-Focused | Replacing existing features |
| 9 | Domain-Driven | Complex domains |
| 10 | Event-Driven | Workflows, notifications |

### Test Focuses (Phase 4 TDD / Phase 7 Feature)
| # | Focus | What it covers |
|---|-------|----------------|
| 1 | Happy Path | Normal usage, core functionality |
| 2 | Edge Cases | Boundaries, empty/null, max values |
| 3 | Error Handling | Invalid inputs, exceptions |
| 4 | Integration | External services, database, APIs |
| 5 | State & Mutations | State management, data transformations |
| 6 | Security | Auth, permissions, input safety |

### Review Focuses (Phase 8)
| # | Focus | Agent | What it covers |
|---|-------|-------|----------------|
| 1 | Correctness & Bugs | `code-reviewer` | Logic errors, null handling, race conditions |
| 2 | Conventions & Style | `code-reviewer` | Project guidelines, naming, patterns |
| 3 | Error Handling | `code-reviewer` | Missing catches, recovery, error messages |
| 4 | Security | `security-auditor` | OWASP Top 10, threat modeling |
| 5 | Performance | `code-reviewer` | Algorithms, memory, caching |
| 6 | Maintainability | `code-reviewer` | Duplication, clarity, testability |

## Usage

```bash
# /feature - implementation first
/feature Add user authentication with OAuth support
/feature  # interactive mode

# /tdd - test first
/tdd Add user authentication with OAuth support
/tdd  # interactive mode

# /code-review - standalone review
/code-review              # review staged changes (default)
/code-review --unstaged   # review working directory changes
/code-review --pr 123     # review pull request
/code-review --commit abc # review specific commit
/code-review src/auth.ts  # review specific files
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

### Use `/code-review` for:
- Quick code review without full workflow overhead
- Reviewing PRs or commits before merge
- Spot-checking staged changes before commit
- Security audits on specific files

### Skip workflows for:
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
