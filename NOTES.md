# Development Notes

## Architecture Approval Hook (hooks/hooks.json)

The current hook uses a prompt-based approach to detect if the user selected an architecture via `AskUserQuestion`.

**If this proves unreliable**, switch to state file approach:
- Phase 4 writes `claude-tmp/devflow-approved.tmp` when user makes selection
- Hook uses `type: "bash"` to check if file exists
- Phase 8 cleans up the temp file

## Workflow State Persistence (Compaction Recovery)

The workflow uses a state file (`claude-tmp/devflow-feature-state.json`) to persist progress across:
- Conversation compaction (automatic summarization when context gets long)
- Session interruptions

### Implementation Details

1. **PreCompact Hook**: Before compaction, writes current phase state to the file
2. **Workflow Start Check**: Command checks for existing state file to detect resumed workflows
3. **Phase Transitions**: State file updated at start/end of each phase
4. **Cleanup**: State file deleted on Phase 8 completion

### Auto-Approval Hook for State File Operations

A PreToolUse hook automatically approves Write/Edit/Bash operations on the state file without requiring user permission:

**Hook Configuration** (hooks/hooks.json):
- Matcher: `Write|Edit|Bash` (positioned first in PreToolUse array)
- Checks if operation targets `claude-tmp/devflow-feature-state.json`
- Auto-approves matching operations
- Allows non-matching operations to pass through to subsequent hooks

**Rationale**: State file operations happen frequently (workflow start, phase transitions, compaction) and shouldn't interrupt user workflow with permission prompts.

### State File Schema

The state file includes full historical context via `phaseHistory`:

```json
{
  "active": true,
  "workflowType": "feature",
  "featureDescription": "Add user authentication",
  "startedAt": "2026-01-13T10:30:00Z",
  "lastUpdatedAt": "2026-01-13T11:45:00Z",
  "currentPhase": 4,
  "currentTask": null,
  "phaseHistory": [
    {
      "phase": 1,
      "name": "Discovery",
      "status": "completed",
      "startedAt": "2026-01-13T10:30:00Z",
      "completedAt": "2026-01-13T10:35:00Z",
      "outputs": {
        "requirements": ["OAuth login", "session management"],
        "constraints": ["Must use existing middleware"]
      }
    },
    {
      "phase": 2,
      "name": "Codebase Exploration",
      "status": "completed",
      "startedAt": "2026-01-13T10:35:00Z",
      "completedAt": "2026-01-13T10:50:00Z",
      "outputs": {
        "agentCount": 3,
        "keyFiles": ["src/auth/middleware.ts"],
        "patterns": ["Express middleware"],
        "integrationPoints": ["src/app.ts"]
      }
    },
    {
      "phase": 3,
      "name": "Clarifying Questions",
      "status": "completed",
      "startedAt": "2026-01-13T10:50:00Z",
      "completedAt": "2026-01-13T11:00:00Z",
      "outputs": {
        "clarifications": [
          {"question": "Which OAuth providers?", "answer": "Google and GitHub"}
        ]
      }
    },
    {
      "phase": 4,
      "name": "Architecture Design",
      "status": "in_progress",
      "startedAt": "2026-01-13T11:00:00Z",
      "completedAt": null,
      "outputs": {}
    }
  ],
  "decisions": {
    "architecture": null,
    "testStrategy": null
  },
  "summary": "Exploring architecture options for OAuth integration."
}
```

### Phase-Specific Outputs

| Phase | Outputs |
|-------|---------|
| 1 Discovery | `requirements[]`, `constraints[]` |
| 2 Exploration | `agentCount`, `keyFiles[]`, `patterns[]`, `integrationPoints[]` |
| 3 Clarifications | `clarifications[]` (question/answer pairs) |
| 4 Architecture | `agentCount`, `optionsPresented[]`, `selectedArchitecture`, `selectionRationale` |
| 5 Planning | `taskCount`, `testCount`, `reviewCount`, `planFile` |
| 6 Implementation | `tasksCompleted[]`, `tasksRemaining[]`, `filesModified[]` |
| 7 Testing | `testsWritten[]`, `testsPassing` |
| 8 Review | `issuesFound`, `issuesFixed[]`, `issuesSkipped[]` |
| 9 Summary | `filesCreated[]`, `filesModified[]`, `testCoverage` |

### Recovery Behavior

When resuming:
1. User informed which phase is being resumed
2. Historical context from `phaseHistory` is displayed:
   - Key files and patterns from exploration
   - Clarifications made
   - Architecture selection and rationale
3. Workflow continues from current phase (no restart)
