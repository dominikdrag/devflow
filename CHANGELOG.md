# Changelog

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
- `/feature-dev` command for guided feature development
- Acknowledgement linking to Anthropic's official feature-dev plugin
