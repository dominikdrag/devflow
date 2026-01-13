# Changelog

## [2.2.4] - 2026-01-13

### Changed
- Clarified in Phase 7 documentation that TEST tasks are added to plan file after user approval

## [2.2.3] - 2026-01-13

### Changed
- **TEST tasks now created dynamically in Phase 7** instead of being pre-planned in Phase 5
  - `test-analyzer` proposes tests based on actual implementation context
  - TEST-NNN tasks created in plan file only after user approves the testing strategy
  - More accurate test proposals since they're based on what was actually built
- Phase 5 (Planning) no longer includes "Testing Tasks" section in plan template
- Phase 7 restructured: analyze → present → approve → create TEST tasks → execute → run

## [2.2.2] - 2026-01-13

### Changed
- Plan files now contain FULL details from phases 1-4/5 (not summaries)
  - Discovery: All requirements and constraints
  - Codebase Exploration: Key files, patterns, integration points, full agent findings
  - Clarifications: Complete Q&A table with all ambiguities resolved
  - Architecture Design: All options considered, selected architecture with full rationale
  - (TDD only) Test Planning: All test cases and strategy details
- Plan files now serve as complete reference enabling workflow resumption

## [2.2.1] - 2026-01-13

### Added
- **`phaseHistory` array in state files** for richer workflow resumption context
  - Each phase records: phase number, name, status, timestamps, and structured outputs
  - Phase-specific outputs preserve key decisions (requirements, patterns, clarifications, architecture)
  - Enables intelligent resumption after compaction with full historical context
- PreCompact hook updated to build complete `phaseHistory` from conversation
- Workflow resumption now displays historical context from completed phases

## [2.2.0] - 2026-01-11

### Added
- **`/tdd` command** - New 9-phase Test-Driven Development workflow with Red-Green-Refactor cycles
  - Test-first approach: tests are designed BEFORE architecture (Phase 4 before Phase 5)
  - `tdd-test-planner` agent (Opus) designs tests from requirements before code exists
  - Per-task Red-Green-Refactor implementation cycles:
    - **RED**: Write failing test, verify it fails
    - **GREEN**: Write minimal code to pass (max 3 retries)
    - **REFACTOR**: Optional cleanup while keeping tests green
  - Separate state files (`tdd-state.json`, `tdd-plan.md`) from `/feature` workflow
  - TDD-specific task structure with `TDD-NNN` tasks and substeps (`TDD-NNN-RED`, `TDD-NNN-GREEN`, `TDD-NNN-REFACTOR`)
  - Configurable agent counts: `--explorers`, `--planners`, `--architects`, `--reviewers`

## [2.1.8] - 2026-01-09

### Changed
- Out-of-range agent count flags now clamp to valid range instead of using defaults
  - e.g., `--explorers=15` now uses 10 (max) instead of 3 (default)
  - More intuitive behavior that respects user intent

## [2.1.7] - 2026-01-08

### Removed
- PreToolUse hook for workflow file auto-approval
  - Permission rules in settings now handle state file operations
  - Simplifies hook configuration to only PreCompact

## [2.1.6] - 2026-01-08

### Changed
- Replaced PreToolUse Write|Edit hook with explicit gate verification in Phase 6
  - Removed prompt-based hook that checked for architecture selection (unreliable)
  - Added "Critical Gates" checklist to Phase 6 requiring verification of both architecture selection and plan approval before implementation

## [2.1.5] - 2026-01-07

### Added
- Explicit phase-task type scope rules documented in workflow and plan file
  - Phase 6 (Implementation) works ONLY on `TASK-NNN` tasks
  - Phase 7 (Testing) works ONLY on `TEST-NNN` tasks
  - Phase 8 (Review) works ONLY on `REVIEW-NNN` tasks
- Phase 7 now reconciles test-analyzer output with existing TEST-NNN tasks
  - Compares analyzer proposals against planned testing tasks
  - Identifies gaps, redundancies, and refinements
  - Updates plan file with refined TEST-NNN tasks after user approval
- Phase 8 now reconciles code-reviewer output with existing REVIEW-NNN tasks
  - Maps high-confidence issues to actionable review tasks
  - Updates plan file with specific issue-fixing tasks after user selection

### Changed
- Plan file template now includes "Phase Scope Rules" section
- Phase 7 restructured into 8 steps with explicit task reconciliation workflow
- Phase 8 restructured into 9 steps with explicit task reconciliation workflow

## [2.1.4] - 2026-01-07

### Changed
- Increased `--explorers` flag maximum from 5 to 10
  - Allows more thorough codebase exploration for large or complex projects
  - Other agent limits remain at 1-5

## [2.1.3] - 2026-01-06

### Changed
- Phases 7-9 now explicitly update the plan file (`claude-tmp/devflow-plan.md`)
  - Phase 7 (Testing): Marks TEST-NNN tasks complete, updates progress log
  - Phase 8 (Quality Review): Tracks review initiation, fix selection, and completion
  - Phase 9 (Summary): Updates status header and acceptance criteria
- Plan file is now only deleted if ALL tasks and acceptance criteria are complete
  - Incomplete workflows preserve the plan file as a record of unfinished work

## [2.1.2] - 2026-01-06

### Changed
- Phase 5 (Planning) now presents the full plan to the user instead of a summary
  - Consistent with Phase 4 and Phase 7 which already show full agent output
  - Ensures maximum user visibility into tasks, dependencies, and acceptance criteria before approval

## [2.1.1] - 2026-01-06

### Changed
- Moved state files from `.claude/` to `claude-tmp/` directory
  - `claude-tmp/devflow-state.json` - workflow state persistence
  - `claude-tmp/devflow-plan.md` - implementation plan
  - Enables auto-edit permission rules in `.claude/settings.local.json`
- Updated hooks to recognize new state file paths

## [2.1.0] - 2026-01-05

### Added
- **New Phase 5: Planning** - Creates implementation plan before coding begins
  - Consolidates outputs from phases 1-4 into a comprehensive plan
  - Defines discrete tasks with IDs (`TASK-NNN`, `TEST-NNN`, `REVIEW-NNN`)
  - Defines acceptance criteria (`AC-NNN`)
  - Writes plan to `.claude/devflow-plan.md` (Markdown for human readability)
  - Requires explicit user approval before implementation begins
- Task-level progress tracking throughout implementation
  - Plan file tracks completed tasks with checkboxes (`- [x]`)
  - Progress log records timestamps of key events
- Enhanced workflow resumption after compaction
  - PreCompact hook now updates plan file progress
  - Resume logic displays current task and remaining count

### Changed
- Workflow expanded from 8 to 9 phases (Planning inserted after Architecture Design)
- Implementation phase now reads and updates the plan file
- PreCompact hook timeout increased from 60s to 90s to accommodate plan file updates
- Phase numbering shifted: Implementation→6, Testing→7, Quality Review→8, Summary→9
- Summary phase now deletes both state file and plan file

## [2.0.0] - 2026-01-05

### Changed
- **BREAKING**: Renamed plugin from `feature-dev` to `devflow` to avoid conflict with Anthropic's official plugin
- **BREAKING**: Renamed command from `/feature-dev` to `/feature`
- Renamed state file from `.claude/feature-dev-state.json` to `.claude/devflow-state.json`
- Updated all internal references and documentation

### Migration
- Users should reinstall: `/plugin install devflow@dominikdrag-marketplace`
- Update any scripts or documentation referencing `/feature-dev` to `/feature`

## [1.1.8] - 2026-01-05

### Changed
- Agent outputs in key decision phases must now be presented in full (not summarized)
  - Phase 4 (Architecture Design): Full architect proposals for informed selection
  - Phase 6 (Testing): Full test-analyzer output for strategy approval
  - Phase 7 (Quality Review): Full reviewer findings for issue selection
  - Ensures maximum user control at critical workflow gates

## [1.1.7] - 2026-01-03

### Fixed
- PreCompact hook schema validation error
  - Added missing `hooks` array wrapper in hooks.json
  - Aligns PreCompact structure with PreToolUse hook format

## [1.1.6] - 2026-01-02

### Added
- Auto-approval hook for state file operations (Write, Edit, Delete)
  - State file modifications no longer require user permission
  - Enables seamless workflow state persistence during compaction
  - Positioned first in PreToolUse hook chain to intercept early

## [1.1.5] - 2026-01-01

### Changed
- Phase 7 (Quality Review) now requires explicit user approval before applying fixes
  - Waits for ALL review agents to complete before proceeding
  - Consolidates and deduplicates findings, organized by severity
  - Presents findings summary to user with file:line and confidence scores
  - Uses `AskUserQuestion` with `multiSelect: true` for per-issue selection
  - Applies only the fixes user explicitly selected
  - Asks user if they want to re-run review after fixes are applied

## [1.1.4] - 2026-01-01

### Added
- Workflow state persistence for compaction recovery
  - State file (`.claude/feature-dev-state.json`) tracks current phase, completed phases, and key decisions
  - `PreCompact` hook automatically saves state before conversation compaction
  - Workflow resumes from correct phase after compaction or session interruption
  - State file cleaned up on Phase 8 completion

## [1.1.3] - 2026-01-01

### Fixed
- Phase 2 now enforces waiting for ALL exploration agents before proceeding to Phase 3
  - Added explicit blocking requirement similar to Phase 4's architecture approval gate
  - Ensures clarifying questions are informed by complete codebase exploration results

## [1.1.2] - 2026-01-01

### Added
- `test-runner` agent (Haiku) for executing tests and reporting results
  - Runs specified tests and provides structured summary
  - Reports passed, failed, skipped tests with details
  - Does not fix failures - only executes and reports

### Changed
- Phase 5 (Implementation) now validates existing tests after implementation
  - Runs `test-runner` agent to check related tests still pass
  - Fixes implementation issues before proceeding to testing phase
- Phase 6 (Testing) now uses `test-runner` agent for test execution
  - Replaces direct test running with structured agent-based execution
  - Provides clearer feedback on test failures

## [1.1.0] - 2026-01-01

### Changed
- Testing phase now requires explicit user approval before writing tests
  - Test analysis results presented to user with summary of categories and cases
  - User must confirm via `AskUserQuestion` (proceed, modify scope, or skip)
  - Mirrors architecture approval pattern from Phase 4
- Upgraded `test-analyzer` agent from Sonnet to Opus for higher quality analysis

## [1.0.6] - 2025-12-30

### Changed
- `code-architect` agent now reads CLAUDE.md and style guides for project rules
  - Ensures architectures align with documented project conventions
  - Matches behavior already present in `code-reviewer` agent

## [1.0.5] - 2025-12-30

### Added
- `test-analyzer` agent (Sonnet) for proposing comprehensive test plans
  - Analyzes implemented code and project test patterns
  - Proposes specific test cases with priorities
  - Identifies mocking requirements and setup complexity

### Changed
- Testing phase now uses hybrid approach:
  - `test-analyzer` agent proposes what should be tested
  - Tests written directly by Claude to preserve implementation context
  - Results in better test coverage than either approach alone

## [1.0.4] - 2025-12-30

### Changed
- Testing phase no longer uses a separate `test-writer` subagent
  - Tests are now written directly by Claude to preserve local implementation context
  - Enables better understanding of what was built and how to test it

### Removed
- `test-writer` agent (functionality moved inline to main workflow)

## [1.0.3] - 2025-12-30

### Changed
- Architecture variations are now surfaced when proposals converge
  - Distinguishes three scenarios: divergent, converged with variations, fully converged
  - Minor variations (naming, patterns, error handling) presented as sub-options
  - User can choose between variation points via `AskUserQuestion`

## [1.0.2] - 2025-12-30

### Fixed
- Architecture approval gate now works reliably
  - Phase 4 presents all distinct architecture proposals instead of auto-synthesizing
  - Uses `AskUserQuestion` for explicit user selection
  - Users can choose from proposed architectures or provide custom approach
  - Updated hook prompt to detect explicit selection responses

### Changed
- Updated README to reflect new architecture selection workflow

## [1.0.1] - 2025-12-29

### Added
- `test-writer` agent (Opus) for comprehensive test generation
- `security-auditor` agent (Sonnet) for optional security analysis
- 8-phase workflow (added Testing and Quality Review phases)
- Workflow enforcement hooks for architecture approval gate

### Changed
- Expanded workflow from basic phases to full 8-phase process

## [1.0.0] - 2025-12-29

### Added
- Initial plugin release
- `code-explorer` agent (Sonnet) for codebase analysis
- `code-architect` agent (Opus) for architecture design
- `code-reviewer` agent (Opus) for code review
- `/feature-dev` command for guided feature development (renamed to `/feature` in v2.0.0)
- Acknowledgement linking to Anthropic's official feature-dev plugin
