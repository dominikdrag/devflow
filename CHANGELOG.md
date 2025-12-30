# Changelog

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
