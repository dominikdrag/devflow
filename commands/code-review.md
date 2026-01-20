---
description: Code review with parallel agents and confidence-based filtering
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite
argument-hint: [--staged|--unstaged|--pr N|--commit HASH|file-path...]
---
# Code Review

You are performing a standalone code review. This command is independent of the /feature and /tdd workflows.

---

## Phase 1: Target Identification

**Goal**: Determine exactly what files/changes to review

### Parse Arguments

Arguments: $ARGUMENTS

**Input Modes**:

| Mode | Trigger | Action |
|------|---------|--------|
| Staged changes | `--staged` or no args | `git diff --cached --name-only` |
| Unstaged changes | `--unstaged` | `git diff --name-only` |
| Pull request | `--pr N` | `gh pr view N --json files` + `gh pr diff N` |
| Specific commit | `--commit HASH` | `git diff HASH~1..HASH --name-only` |
| Explicit files | file paths | Use provided paths directly |

**Default behavior** (no arguments):
1. Check if there are staged changes (`git diff --cached --name-only`)
2. If staged changes exist, review those
3. If no staged changes, check unstaged changes (`git diff --name-only`)
4. If no changes at all, ask user what to review

### Actions

1. Parse arguments per the Input Modes table above
2. Run appropriate git/gh command to get file list
3. For diff-based modes, also get the actual diff content:
   - Staged: `git diff --cached`
   - Unstaged: `git diff`
   - PR: `gh pr diff N`
   - Commit: `git show HASH`
4. Filter to code files (exclude images, binaries, lockfiles like package-lock.json, yarn.lock)
5. Display summary:
   ```
   ## Review Target

   **Mode**: [staged changes|unstaged changes|PR #N|commit abc1234|explicit files]
   **Files** ({count}):
   - path/to/file1.ts (modified)
   - path/to/file2.ts (added)
   - path/to/file3.ts (deleted)

   **Lines changed**: +{additions} / -{deletions}
   ```

6. If more than 20 files, warn user and offer to narrow scope
7. Create state file (`claude-tmp/code-review-state.json`):
   ```json
   {
     "sessionId": "review-{timestamp}",
     "mode": "staged|unstaged|pr|commit|files",
     "modeArg": "N|HASH|null",
     "files": ["path/to/file1.ts", "..."],
     "startedAt": "ISO timestamp",
     "reviewCycle": 1,
     "focusesSelected": [],
     "previousIssues": [],
     "fixesApplied": []
   }
   ```

### Validation

- If no files found: Inform user and ask what to review
- If files don't exist: Report error with specific paths
- If PR not found: Display error, suggest checking PR number
- If commit not found: Display error, suggest checking commit hash

---

## Phase 2: Focus Selection

**Goal**: Let user choose which review focuses to apply

**Actions**:

Display available review focuses:

```
## Select Review Focuses

Each selected focus launches a parallel reviewer agent.

  Core Focuses (recommended)
   1. Correctness & Bugs   - Logic errors, null handling, race conditions
   2. Conventions & Style  - Project guidelines, naming, patterns
   3. Error Handling       - Missing catches, recovery, error messages

  Specialized Focuses
   4. Security             - For: auth, user input, external APIs, sensitive data
   5. Performance          - For: hot paths, data processing, resource management
   6. Maintainability      - For: large changes, shared code, public APIs

Enter selection (e.g., "1,2,3" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Parse Selection

Accept input formats:
- Comma-separated: `1,2,3`
- Ranges: `1-3`
- Mixed: `1-3,5`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-6 only, deduplicate, sort.

Update state file with `focusesSelected`.

---

## Phase 3: Launch Review Agents

**Goal**: Run parallel review agents based on selected focuses

**Actions**:

Launch agents based on selected focuses, all in parallel:
- **Focuses 1, 2, 3, 5, 6** → Launch `code-reviewer` agent with focus injected
- **Focus 4 (Security)** → Launch `security-auditor` agent (already specialized)

**Agent Prompt Format for code-reviewer**: Include the focus assignment in each agent's prompt:
> Your assigned review focus is: **[Focus Name]**
>
> [Include the FULL focus definition from the reference below]
>
> Review the following code changes:
>
> **Files to review**: [list files]
>
> **Diff context** (if available):
> ```diff
> [relevant diff content]
> ```

**Focus Definitions for code-reviewer** (inject the selected one into each agent prompt):

1. **Correctness & Bugs**
   - Focus: Logic errors and functional correctness
   - When to use: Always - catches bugs that affect functionality
   - Logic errors and incorrect conditions
   - Null/undefined handling issues
   - Off-by-one errors
   - Race conditions and concurrency bugs
   - Resource leaks (memory, file handles, connections)

2. **Conventions & Style**
   - Focus: Project guidelines and code consistency
   - When to use: Always - ensures codebase consistency
   - Adherence to CLAUDE.md and style guides
   - Import conventions and module organization
   - Naming conventions
   - Framework-specific patterns
   - Code organization and structure

3. **Error Handling**
   - Focus: Failure modes and error recovery
   - When to use: Always - ensures graceful failure behavior
   - Missing try/catch blocks where needed
   - Inadequate error recovery strategies
   - Unclear or missing error messages
   - Cleanup and resource release on errors
   - Error propagation patterns

5. **Performance**
   - Focus: Efficiency and resource usage
   - When to use: Hot paths, data processing, loops, resource-constrained code
   - Inefficient algorithms (O(n²) where O(n) possible)
   - N+1 query problems
   - Memory leaks and excessive allocations
   - Missing caching opportunities
   - Blocking operations in async contexts

6. **Maintainability**
   - Focus: Long-term code health and readability
   - When to use: Large changes, shared utilities, public APIs
   - Code duplication that should be abstracted
   - Unclear or misleading code
   - Missing or outdated documentation
   - Testability concerns
   - Excessive complexity

**For Focus 4 (Security)**: Launch `security-auditor` agent directly (it's already specialized for OWASP Top 10, threat modeling, etc.). No focus injection needed - just provide the files and diff.

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

---

## Phase 4: Present Findings

**Goal**: Show complete review results to user

**Actions**:

### Step 1: Present Full Agent Output

**IMPORTANT**: Present the FULL output from each review agent to the user - do NOT summarize or condense their findings. The user needs complete visibility into each reviewer's analysis, reasoning, and specific concerns to make informed decisions about which issues to address.

### Step 2: Organize by Severity

Map issues based on confidence thresholds:
- **code-reviewer**: Report issues with confidence >= 80/100
- **security-auditor**: Report issues with confidence >= 85/100

Deduplicate overlapping issues (same file + line + similar description).

Organize by severity:
- **Critical Issues** (Confidence 90-100): Must be fixed
- **Important Issues** (Confidence 80-89): Should be addressed
- **Suggestions** (Confidence < 80): Nice to have improvements

### Step 3: Display Organized Summary

```
## Review Findings Summary

### Critical Issues ({count})
1. [FILE:LINE] Description - {confidence}%
   - Why: [impact]
   - Fix: [suggestion]
2. ...

### Important Issues ({count})
1. [FILE:LINE] Description - {confidence}%
   - Why: [impact]
   - Fix: [suggestion]
2. ...

### Suggestions ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...
```

### Step 4: Update State File

Update `claude-tmp/code-review-state.json` with `previousIssues` array:
```json
{
  "previousIssues": [
    {"id": 1, "file": "...", "line": N, "description": "...", "confidence": N, "severity": "critical|important|suggestion", "focus": "..."}
  ]
}
```

---

## Phase 5: Fix Selection

**Goal**: Let user choose which issues to address

**Actions**:

Ask user which issues to fix:

```
Which issues should I fix?

Enter numbers separated by commas (e.g., "1,3,5"), "critical" for critical only, "all" to fix everything, or "skip" to finish:
```

Parse user response:
- **Numbers**: Select specific issues by their displayed number
- **"critical"**: Select all critical issues (confidence 90-100)
- **"all"**: Select all issues (critical + important + suggestions)
- **"skip"**: Proceed to summary without fixing

If user selects "skip", go directly to Phase 8 (Summary).

---

## Phase 6: Apply Fixes

**Goal**: Apply selected fixes

**Actions**:

For each selected issue:
1. Read the file
2. Apply the fix based on the reviewer's suggestion
3. Track in state file: add to `fixesApplied` array with `{issueId, timestamp}`

After all fixes:
1. Update state file
2. Display summary:
   ```
   ## Fixes Applied

   - [x] Issue 1: [description]
   - [x] Issue 3: [description]
   - [x] Issue 5: [description]

   {count} issues fixed.
   ```

---

## Phase 7: Re-review Option

**Goal**: Offer to verify fixes with another review cycle

**Actions**:

If any fixes were applied, ask:

```
Would you like to re-run the review to verify fixes?

Type: "review" to run again (same or different focuses), or "done" to finish:
```

**If user types "review"**:
1. Increment `reviewCycle` in state file
2. Return to Phase 2 (Focus Selection)
3. After new review, compare with `previousIssues`:
   - Note fixed issues (no longer appearing)
   - Note new issues (appeared after fixes)
   - Note remaining issues (still present)

**If user types "done"**:
1. Proceed to Phase 8 (Summary)

---

## Phase 8: Summary

**Goal**: Provide final review summary and cleanup

**Actions**:

Display final summary:

```
## Code Review Complete

**Target**: [mode description - e.g., "staged changes" or "PR #123"]
**Files reviewed**: {fileCount}
**Review cycles**: {reviewCycle}
**Issues found**: {totalIssuesFound}
**Issues fixed**: {issuesFixedCount}
**Issues skipped**: {issuesSkippedCount}

### Session Log
- Cycle 1: Found {N} issues, fixed {M}
- Cycle 2: Found {N} issues (verification), fixed {M}
```

If any issues remain:
```
### Remaining Issues
- [FILE:LINE] Description (skipped)
```

Cleanup:
- Delete `claude-tmp/code-review-state.json`

---

## Error Handling

| Scenario | Handling |
|----------|----------|
| No staged/unstaged changes | Ask user to specify files or provide a different mode |
| PR not found | Display error with `gh pr view` output, suggest checking PR number |
| Commit not found | Display error, suggest checking commit hash with `git log --oneline -10` |
| No files to review after filtering | Inform user, suggest different target |
| Too many files (>20) | Warn and offer to narrow scope or proceed |
| Agent timeout | Report partial results, offer to retry specific agents |

---

## Usage Examples

```bash
# Review staged changes (default)
/code-review

# Review unstaged changes
/code-review --unstaged

# Review a pull request
/code-review --pr 123

# Review a specific commit
/code-review --commit abc1234

# Review specific files
/code-review src/auth.ts src/api/users.ts

# Review a directory
/code-review src/components/
```
