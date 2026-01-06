# Development Notes

## Architecture Approval Hook (hooks/hooks.json)

The current hook uses a prompt-based approach to detect if the user selected an architecture via `AskUserQuestion`.

**If this proves unreliable**, switch to state file approach:
- Phase 4 writes `claude-tmp/devflow-approved.tmp` when user makes selection
- Hook uses `type: "bash"` to check if file exists
- Phase 8 cleans up the temp file

## Workflow State Persistence (Compaction Recovery)

The workflow uses a state file (`claude-tmp/devflow-state.json`) to persist progress across:
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
- Checks if operation targets `claude-tmp/devflow-state.json`
- Auto-approves matching operations
- Allows non-matching operations to pass through to subsequent hooks

**Rationale**: State file operations happen frequently (workflow start, phase transitions, compaction) and shouldn't interrupt user workflow with permission prompts.

### State File Schema

```json
{
  "active": true,
  "currentPhase": 1,
  "completedPhases": [1, 2],
  "featureDescription": "Add user authentication",
  "decisions": {
    "architecture": "Option A: JWT-based auth",
    "testStrategy": "Proceed with proposed testing"
  },
  "summary": "Completed exploration and clarifying questions. Ready for architecture design."
}
```

### Recovery Behavior

When resuming:
1. User informed which phase is being resumed
2. Completed phases and key decisions are displayed
3. Workflow continues from current phase (no restart)
