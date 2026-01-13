---
description: Guided feature development with codebase understanding and architecture focus
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite, AskUser
argument-hint: [--explorers=N] [--architects=N] [--analyzers=N] [--reviewers=N] <feature-description>
---

# Feature Development Workflow

You are guiding the user through a systematic 9-phase feature development process. This workflow ensures deep codebase understanding before implementation.

## Core Principles

1. **Ask clarifying questions** - Identify ambiguities early, before designing architecture
2. **Understand before building** - Use agents to explore the codebase thoroughly
3. **Read what agents find** - Always read the files identified by exploration agents
4. **Keep code simple** - Follow existing patterns, avoid over-engineering

---

## Workflow State Management

This workflow uses a state file (`claude-tmp/devflow-state.json`) to persist progress across conversation compaction and session interruptions.

### On Workflow Start

**FIRST**, check if `claude-tmp/devflow-state.json` exists:
- **If file exists and `active: true`**: This is a RESUMED workflow
  - Read the state file to understand current progress
  - **If `claude-tmp/devflow-plan.md` exists**: Read the plan file
    - Identify the current task from `currentTask` in state file
    - Count remaining unchecked tasks (lines matching `- [ ]`)
    - Display: "Current task: {currentTask}, {N} tasks remaining"
  - Inform the user: "Resuming devflow workflow from Phase {currentPhase}"
  - Display historical context from `phaseHistory`:
    - For each completed phase, show phase name and key outputs
    - Phase 2: Show key files and patterns discovered
    - Phase 3: Show clarifications made
    - Phase 4: Show selected architecture and rationale
  - Continue from the current phase (do NOT restart from Phase 1)
- **If file does not exist**: This is a NEW workflow
  - Create initial state file:
    ```json
    {
      "active": true,
      "workflowType": "feature",
      "featureDescription": "[from user input]",
      "startedAt": "[current ISO timestamp]",
      "lastUpdatedAt": "[current ISO timestamp]",
      "currentPhase": 1,
      "currentTask": null,
      "phaseHistory": [],
      "decisions": {"architecture": null, "testStrategy": null},
      "summary": "Starting feature development workflow"
    }
    ```
  - Proceed with Phase 1

### State File Format

```json
{
  "active": true,
  "workflowType": "feature",
  "featureDescription": "...",
  "startedAt": "ISO timestamp",
  "lastUpdatedAt": "ISO timestamp",
  "currentPhase": 1,
  "currentTask": null,
  "phaseHistory": [
    {
      "phase": 1,
      "name": "Discovery",
      "status": "completed",
      "startedAt": "ISO timestamp",
      "completedAt": "ISO timestamp",
      "outputs": {
        "requirements": ["requirement 1", "requirement 2"],
        "constraints": ["constraint 1"]
      }
    }
  ],
  "decisions": {
    "architecture": null,
    "testStrategy": null
  },
  "summary": "Brief context for resumption"
}
```

### Phase-Specific Outputs

Each phase stores structured outputs in `phaseHistory[].outputs`:

| Phase | Name | Outputs |
|-------|------|---------|
| 1 | Discovery | `requirements[]`, `constraints[]` |
| 2 | Codebase Exploration | `agentCount`, `keyFiles[]`, `patterns[]`, `integrationPoints[]` |
| 3 | Clarifying Questions | `clarifications[]` (array of `{question, answer}`) |
| 4 | Architecture Design | `agentCount`, `optionsPresented[]`, `selectedArchitecture`, `selectionRationale` |
| 5 | Planning | `taskCount`, `testCount`, `reviewCount`, `planFile` |
| 6 | Implementation | `tasksCompleted[]`, `tasksRemaining[]`, `filesModified[]` |
| 7 | Testing | `testsWritten[]`, `testsPassing` |
| 8 | Quality Review | `issuesFound`, `issuesFixed[]`, `issuesSkipped[]` |
| 9 | Summary | `filesCreated[]`, `filesModified[]`, `testCoverage` |

### Updating State

At the START of each phase, update the state file:
- Set `currentPhase` to the new phase number
- Set `lastUpdatedAt` to current ISO timestamp
- Add new entry to `phaseHistory[]` with `status: "in_progress"`, `startedAt`, and empty `outputs`
- Update `summary` with relevant context

At the END of each phase, update the state file:
- Update the phase's `phaseHistory` entry: set `status: "completed"`, `completedAt`, and populate `outputs`
- Store any decisions made (architecture selection, test strategy approval) in `decisions`

---

## Configuration

### Parse Arguments

Arguments: $ARGUMENTS

Parse optional flags to configure agent counts:
- `--explorers=N` - Number of `code-explorer` agents (default: 3)
- `--architects=N` - Number of `code-architect` agents (default: 3)
- `--analyzers=N` - Number of `test-analyzer` agents (default: 1)
- `--reviewers=N` - Number of `code-reviewer` agents (default: 3)

Valid range: 1-10 for explorers, 1-5 for all others. If a value is out of range, use the closest value in range.

Remaining text after flags is the feature description.

### Display Configuration

At the start, confirm the configuration:
> Using agent counts: {explorers} explorers, {architects} architects, {analyzers} analyzers, {reviewers} reviewers

---

## Phase 1: Discovery

**Goal**: Understand what needs to be built

**Actions**:
1. If the user provided a feature description, summarize your understanding
2. Identify the core requirements and constraints
3. Ask initial clarifying questions if the request is ambiguous
4. Confirm understanding before proceeding

**Output**: Clear statement of what will be built

---

## Phase 2: Codebase Exploration

**Goal**: Understand existing patterns and relevant code

**Actions**:
1. Launch {explorers} `code-explorer` agent(s) in parallel:
   - **If 1**: Focus on primary integration points and similar features
   - **If 2**: Split between (1) similar features and (2) integration points
   - **If 3+**: Distribute across similar features, related subsystems, and integration points
2. **WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned
3. Read the essential files identified by the agents
4. Synthesize findings into a codebase understanding summary

**CRITICAL**: You MUST wait for ALL exploration agents to return their complete output before proceeding to Phase 3. The exploration results are essential for asking informed clarifying questions. NEVER proceed to Phase 3 while agents are still running or before reviewing their findings.

**Output**: Summary of relevant patterns, conventions, and integration points

---

## Phase 3: Clarifying Questions

**Goal**: Resolve all ambiguities through iterative dialogue before designing

**Approach**: Use progressive clarification rounds rather than asking all questions at once. User answers often reveal new considerations that require follow-up.

**Actions**:
1. **Initial Assessment**: Based on codebase exploration, identify the most critical gaps:
   - Edge cases not covered
   - Integration decisions
   - UX/behavior choices
   - Performance requirements
   - Error handling expectations
   - Scope boundaries

2. **Clarification Loop** (repeat until confident):
   a. Present focused questions for the current round (prioritize by impact)
   b. Wait for user answers
   c. Analyze answers:
      - Update understanding of requirements
      - Note any new ambiguities revealed by answers
      - Identify follow-up questions triggered by responses
   d. Re-assess scope:
      - Are core requirements now well-defined?
      - Are integration points clear?
      - Are edge cases understood?
      - Are there remaining high-impact unknowns?
   e. **Confidence check**: Proceed if:
      - No critical ambiguities remain
      - Answers are internally consistent
      - Scope is bounded and achievable
      - Integration approach is clear
   f. If not confident, continue loop with follow-up questions

3. **Exit Criteria** - Proceed to Phase 4 when ALL of these are true:
   - Core functionality is well-specified
   - Integration points are identified and understood
   - Known edge cases have defined handling
   - No answers contradict each other
   - Scope creep risks are addressed

4. Summarize the final understanding before proceeding

**Output**: Complete requirements with all ambiguities resolved through iterative dialogue

---

## Phase 4: Architecture Design

**Goal**: Design the implementation approach with explicit user selection

**Actions**:
1. Launch {architects} `code-architect` agent(s) with different focuses:
   - **If 1**: Single comprehensive architecture covering all aspects
   - **If 2**: Split between (1) core implementation and (2) integration/edge cases
   - **If 3+**: Distribute across core implementation, integration, and edge cases/error handling
2. When agents complete, analyze their proposals for similarity:
   - Compare: file structures, component breakdowns, data flows, technology choices
   - **If proposals diverge significantly**: Present as DISTINCT architecture options
   - **If proposals converge with minor variations**: Present the converged approach AND highlight variations as sub-options (e.g., naming conventions, specific patterns, error handling strategies)
   - **If proposals fully converge**: Present unified approach with note that agents agreed
3. Present architecture(s) to the user:
   - **For divergent options**: Present each with name, key decisions, files, pros/cons
   - **For converged with variations**: Present unified core approach, then list each variation point with the different approaches proposed
   - **For fully converged**: Present unified approach
4. Use `AskUserQuestion` tool to get EXPLICIT selection:
   - **For divergent options**: Offer each architecture as a numbered option
   - **For converged with variations**: Present the converged approach, then ask user to choose between the specific variation points (may require multiple questions)
   - **ALWAYS** include "Custom: I'll describe my own approach" as final option
5. If user selects custom approach:
   - Wait for user to describe their approach
   - Summarize and confirm with another `AskUserQuestion`
6. Document the selected architecture before proceeding

**CRITICAL**: Do NOT proceed to Phase 5 until user has made an explicit selection via `AskUserQuestion`. The response IS the approval gate.

**IMPORTANT**: Present the FULL output from architecture agents to the user - do NOT summarize or condense their proposals. The user needs complete visibility into each architect's reasoning, trade-offs, and implementation details to make an informed decision. This is a key decision point requiring maximum user control.

**Output**: User-selected architecture blueprint

---

## Phase 5: Planning

**Goal**: Create a comprehensive implementation plan before coding begins

**Actions**:
1. **Consolidate Information**: Gather all outputs from Phases 1-4:
   - Feature requirements and constraints (Discovery)
   - Codebase patterns and integration points (Exploration)
   - Clarification decisions (Q&A)
   - Selected architecture with rationale

2. **Define Implementation Tasks**: Break down the work into discrete, actionable tasks:
   - Create tasks for each file to be created/modified
   - Group tasks by phase (Implementation, Quality)
   - Establish task dependencies where needed
   - Assign task IDs: `TASK-NNN` for implementation, `REVIEW-NNN` for quality
   - **Note**: `TEST-NNN` tasks are created dynamically in Phase 7 by the test-analyzer

   **Phase Scope Rules** (document explicitly in plan):
   - **Phase 6 (Implementation)**: Works ONLY on `TASK-NNN` tasks
   - **Phase 7 (Testing)**: Creates `TEST-NNN` tasks from test-analyzer output, then executes them
   - **Phase 8 (Review)**: Works ONLY on `REVIEW-NNN` tasks

3. **Define Acceptance Criteria**: Extract or derive acceptance criteria from requirements
   - Each criterion should be testable/verifiable
   - Assign IDs: `AC-NNN`

4. **Write Plan File**: Create `claude-tmp/devflow-plan.md` with this structure.

   **IMPORTANT**: Include FULL details from phases 1-4, not summaries. The plan file should be a complete reference that enables resumption without needing the state file.

   ```markdown
   # Feature Development Plan

   > **Status**: In Progress
   > **Current Task**: [none]
   > **Created**: [ISO timestamp]
   > **Last Updated**: [ISO timestamp]

   ## Phase 1: Discovery

   ### Requirements
   - [Full list of ALL requirements identified]

   ### Constraints
   - [Full list of ALL constraints identified]

   ### Initial Understanding
   [Complete statement of what will be built - the full output from Phase 1]

   ## Phase 2: Codebase Exploration

   ### Key Files
   - `path/to/file.ts` - [why it's relevant, what patterns it contains]
   - `path/to/file2.ts` - [why it's relevant, what patterns it contains]

   ### Patterns Discovered
   - [Pattern 1]: [full description and where/how it's used]
   - [Pattern 2]: [full description and where/how it's used]

   ### Integration Points
   - [Integration point 1]: [files involved, how to integrate, any considerations]
   - [Integration point 2]: [files involved, how to integrate, any considerations]

   ### Agent Findings Summary
   [Full synthesis of what exploration agents discovered - preserve all relevant details]

   ## Phase 3: Clarifications

   ### Questions & Answers
   | Question | Answer |
   |----------|--------|
   | [Question 1] | [User's complete answer] |
   | [Question 2] | [User's complete answer] |

   ### Resolved Ambiguities
   [List of all ambiguities that were resolved through Q&A, with the resolution]

   ## Phase 4: Architecture Design

   ### Options Considered
   [Full description of ALL architecture options that were presented to the user]

   ### Selected Architecture
   **Choice**: [Selected option name]

   **Rationale**: [Complete rationale for why this was chosen]

   **Key Decisions**:
   - [Decision 1 with full context]
   - [Decision 2 with full context]

   **Files to Create/Modify**:
   - `path/to/new/file.ts` - [purpose and what it will contain]
   - `path/to/existing/file.ts` - [what specific changes will be made]

   ## Phase Scope Rules
   - **Phase 6 (Implementation)**: Works ONLY on `TASK-NNN` tasks
   - **Phase 7 (Testing)**: Creates `TEST-NNN` tasks from test-analyzer, then executes them
   - **Phase 8 (Review)**: Works ONLY on `REVIEW-NNN` tasks

   ## Implementation Tasks
   - [ ] **TASK-001**: [description]
     - Files: `path/to/file.ts`
     - Depends on: none
   - [ ] **TASK-002**: [description]
     - Files: `path/to/file.ts`
     - Depends on: TASK-001

   ## Quality Tasks
   - [ ] **REVIEW-001**: Run code reviewers
   - [ ] **REVIEW-002**: Apply selected fixes

   ## Acceptance Criteria
   - [ ] **AC-001**: [criterion]
   - [ ] **AC-002**: [criterion]

   ## Progress Log
   | Timestamp | Event |
   |-----------|-------|
   | [ISO] | Planning phase completed |
   ```

5. **Present Full Plan**: Display the complete plan content to the user by reading and showing `claude-tmp/devflow-plan.md`

**IMPORTANT**: Present the FULL plan to the user - do NOT summarize or condense. The user needs complete visibility into every task, its dependencies, and acceptance criteria to make an informed approval decision. This is a key decision point requiring maximum user control.

6. **Plan Approval**: Use `AskUserQuestion` with options:
   - "Proceed with this plan"
   - "Modify the plan" (user describes changes)
   - "Add more tasks"

7. If user selects "Modify the plan" or "Add more tasks":
   - Wait for user input
   - Update the plan file accordingly
   - Re-present summary and ask again

8. **Finalize**: Add approval timestamp to progress log

**CRITICAL**: Do NOT proceed to Phase 6 until user explicitly approves the plan via `AskUserQuestion`.

**Output**: `claude-tmp/devflow-plan.md` file ready to guide implementation

---

## Phase 6: Implementation

**Goal**: Build the feature following the approved plan

**Scope**: This phase works ONLY on `TASK-NNN` implementation tasks. Testing tasks (`TEST-NNN`) and review tasks (`REVIEW-NNN`) are handled in their respective phases.

**CRITICAL GATES** (verify before ANY implementation):
- [ ] Architecture selected via `AskUserQuestion` in Phase 4
- [ ] Plan approved via `AskUserQuestion` in Phase 5

If either gate is missing, STOP and complete the required phase first.

**Actions**:
1. **Verify both gates above are satisfied before writing any code**
2. **Read the plan file** (`claude-tmp/devflow-plan.md`) to get the task list
3. For each task in the "Implementation Tasks" section:
   - Update state file with `currentTask: "TASK-NNN"`
   - Read existing similar code for patterns
   - Implement following established conventions
   - Keep changes minimal and focused
   - **Mark task complete in plan file**: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]` line
   - Add progress log entry: `| [timestamp] | TASK-NNN completed |`
4. Use TodoWrite to track implementation progress (mirrors plan file)
5. Commit logical chunks (if user requests commits)
6. **Validate existing tests**:
   - Launch `test-runner` agent to run tests related to the changes
   - If tests fail, fix the implementation before proceeding
   - Re-run tests until all pass

**Guidelines**:
- Follow existing code patterns exactly
- Don't add unrequested features
- Don't refactor unrelated code
- Keep implementations simple
- **Update plan file after each task completion**

**Output**: Implemented feature with passing existing tests, updated plan file

---

## Phase 7: Testing

**Goal**: Ensure comprehensive test coverage with user-approved strategy

**Scope**: This phase creates and executes `TEST-NNN` testing tasks. Implementation tasks (`TASK-NNN`) were completed in Phase 6. Review tasks (`REVIEW-NNN`) are handled in Phase 8.

**Approach**: Launch test-analyzer agent(s) to propose test cases based on the implemented code, get user approval, create TEST-NNN tasks in the plan file, then write tests directly (not delegated) to preserve local context from implementation. Use `test-runner` agent to execute tests.

**Actions**:

### Step 1: Launch Test Analysis
1. Launch {analyzers} `test-analyzer` agent(s) to analyze the implemented changes:
   - **If 1**: Single comprehensive test analysis
   - **If 2+**: Distribute across unit tests, integration tests, and edge cases
   - Each will identify test patterns, propose test cases, and note mocking requirements
2. **Wait for ALL analyzer agents to complete**

### Step 2: Present to User
1. Present the FULL output from test-analyzer agent(s) - do NOT summarize
2. Present the proposed testing strategy:
   - Test cases organized by type (unit, integration, edge cases)
   - Mocking requirements identified
   - Coverage expectations

**IMPORTANT**: Present the FULL output from test-analyzer agent(s) to the user - do NOT summarize or condense the test proposals. The user needs complete visibility into each proposed test case, its rationale, edge cases identified, and mocking requirements to make an informed decision about the testing strategy.

### Step 3: User Approval
Use `AskUserQuestion` to get EXPLICIT confirmation:
- Option 1: "Proceed with proposed testing strategy"
- Option 2: "Modify testing scope" (user describes changes)
- Option 3: "Skip testing phase"

### Step 4: Create TEST Tasks in Plan File
If user approves the testing strategy:
1. Update `claude-tmp/devflow-plan.md`:
   - Add "## Testing Tasks" section after Implementation Tasks
   - Create `TEST-NNN` tasks based on approved test proposals (starting from TEST-001)
   - Each task should specify: test file path, what behavior is tested, edge cases covered
2. Add progress log entry: `| [timestamp] | Testing tasks created from test-analyzer output |`

### Step 5: Execute Testing Tasks
For each TEST-NNN task in the updated plan:
1. Update state file with `currentTask: "TEST-NNN"`
2. Write tests following the task description and project conventions:
   - Happy path scenarios
   - Edge cases and boundary conditions
   - Error handling paths
   - Integration points with existing code
3. Mark task complete: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]`
4. Add progress log entry: `| [timestamp] | TEST-NNN completed |`

### Step 6: Run Tests
1. Launch `test-runner` agent to execute all new tests
2. If tests fail:
   - Review the failure report
   - Fix the failing tests (adjust test code or implementation as needed)
   - Re-run until all pass
3. When all pass: Add `| [timestamp] | Testing phase completed |`

**CRITICAL**: Do NOT write tests until user has approved the testing strategy via `AskUserQuestion`.

**Test Quality Standards**:
- Test names clearly describe what is being tested
- Each test focuses on a single behavior
- Tests are independent (no shared state)
- Follow existing project test patterns exactly

**Output**: User-approved test suite with passing tests, updated plan file with refined TEST-NNN tasks

---

## Phase 8: Quality Review

**Goal**: Ensure code quality and correctness through user-reviewed findings

**Scope**: This phase works ONLY on `REVIEW-NNN` quality review tasks. Implementation tasks (`TASK-NNN`) and testing tasks (`TEST-NNN`) were completed in previous phases.

**Actions**:

### Step 1: Review Existing Review Tasks
1. Read `claude-tmp/devflow-plan.md` and extract all `REVIEW-NNN` tasks
2. Note the current review scope defined during planning

### Step 2: Launch Review Agents
1. Launch {reviewers} `code-reviewer` agent(s) in parallel:
   - **If 1**: Comprehensive review covering all aspects
   - **If 2**: Split between (1) correctness/bugs and (2) conventions/maintainability
   - **If 3+**: Distribute across correctness, conventions, and maintainability
2. **Wait for ALL agents to complete** before proceeding
3. **Optional**: If user requests security audit, also launch `security-auditor` agent and wait

### Step 3: Present Full Agent Output
**IMPORTANT**: Present the FULL output from each review agent to the user - do NOT summarize or condense their findings. The user needs complete visibility into each reviewer's analysis, reasoning, and specific concerns to make informed decisions about which issues to address. This is a critical quality gate requiring maximum user control.

### Step 4: Reconcile with Plan
1. Compare code-reviewer findings against existing `REVIEW-NNN` tasks
2. Map high-confidence issues (>=80) to actionable review tasks
3. Propose updates to `REVIEW-NNN` tasks:
   - Add specific issue-fixing tasks (e.g., "REVIEW-003: Fix null check in auth.ts:45")
   - Refine generic review tasks with specific findings
   - Note any REVIEW-NNN tasks that are no longer relevant
4. Deduplicate overlapping issues (same file + line + similar description)
5. Organize by severity:
   - **Critical Issues** (Confidence 90-100): Must be fixed
   - **Important Issues** (Confidence 80-89): Should be addressed
   - **Suggestions** (Confidence < 80): Nice to have improvements

### Step 5: Present Reconciled Review Tasks
Display:
1. Original REVIEW-NNN tasks from plan
2. Proposed changes based on reviewer findings
3. Consolidated findings summary:

```
## Review Findings Summary

### Original REVIEW Tasks
- REVIEW-001: [description] - [keep/modify/complete]
- REVIEW-002: [description] - [keep/modify/complete]

### Proposed New REVIEW Tasks (from reviewer findings)
- REVIEW-003: [specific issue from findings]
- REVIEW-004: [specific issue from findings]

### Critical Issues ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...

### Important Issues ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...

### Suggestions ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...
```

### Step 6: User Selection
Use `AskUserQuestion` with `multiSelect: true` to let user choose which issues/tasks to address:
- List each issue as a selectable option
- Group by severity in the question
- Include "Skip all - proceed to summary" as an option

### Step 7: Update Plan File
1. Update `claude-tmp/devflow-plan.md`:
   - Add/modify REVIEW-NNN tasks based on user selection
   - Each selected issue becomes a trackable task with sequential ID
   - Mark any skipped original REVIEW-NNN tasks as complete (if user chose to skip)
2. Add progress log entry: `| [timestamp] | Review tasks refined based on reviewer output |`
3. Add progress log entry: `| [timestamp] | User selected {count} issues to fix |`

### Step 8: Execute Review Tasks
For each selected REVIEW-NNN task in the updated plan:
1. Update state file with `currentTask: "REVIEW-NNN"`
2. Apply the fix
3. Mark task complete: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]`
4. Add progress log entry: `| [timestamp] | REVIEW-NNN completed |`

### Step 9: Offer Re-review
If any fixes were applied, use `AskUserQuestion` to ask:
- "Run review again to verify fixes?"
- "Proceed to summary"

If user chooses re-review, return to Step 2 with a focused scope.

**Plan File Updates Summary**:
- At phase start: Add `| [timestamp] | Quality review initiated |` to progress log
- After reconciliation: Add `| [timestamp] | Review tasks refined based on reviewer output |`
- After user selection: Add `| [timestamp] | User selected {count} issues to fix |`
- After each fix: Mark `REVIEW-NNN` task complete with `Completed: [timestamp]}`, add progress log entry
- After re-review (if done): Add `| [timestamp] | Re-review completed |`
- At phase end: Add `| [timestamp] | Quality review phase completed |`
- Update header: `Current Task`, `Last Updated`

**Output**: Quality-verified implementation with updated plan file reflecting all review tasks

---

## Phase 9: Summary

**Goal**: Document what was accomplished and clean up workflow state

**Actions**:
1. List all files created or modified
2. Summarize key architectural decisions made
3. Document test coverage achieved
4. Note any deferred work or known limitations
5. Suggest potential follow-up tasks
6. Mark all todos as complete

**Plan File Updates**:
1. Update header: Set `Status: Complete` (or `Status: Incomplete` if any tasks failed), update `Last Updated`
2. Mark acceptance criteria: Change `- [ ]` to `- [x]` for completed, add notes for incomplete
3. Add final progress log entries:
   - `| [timestamp] | Acceptance criteria reviewed |`
   - `| [timestamp] | Feature development completed |`

**Cleanup** (conditional):
- Delete state file (`claude-tmp/devflow-state.json`)
- **Only delete plan file if ALL tasks and acceptance criteria are complete**
- If any tasks are incomplete, keep `devflow-plan.md` as a record of unfinished work

**Output**: Completion summary with next steps

---

## Usage

This workflow is invoked with `/feature` followed by optional flags and a feature description:

### Basic Usage
```
/feature Add user authentication with OAuth support
/feature
```

### With Agent Count Overrides
```
/feature --explorers=2 --architects=2 Add user authentication
/feature --reviewers=5 Implement payment processing
/feature --explorers=1 --architects=1 --reviewers=1 Small utility function
```

### Flag Reference

| Flag | Default | Range | Phase |
|------|---------|-------|-------|
| `--explorers=N` | 3 | 1-10 | Phase 2: Codebase Exploration |
| `--architects=N` | 3 | 1-5 | Phase 4: Architecture Design |
| `--analyzers=N` | 1 | 1-5 | Phase 7: Testing |
| `--reviewers=N` | 3 | 1-5 | Phase 8: Quality Review |

If no description is provided, ask the user what feature they want to build.

## When to Use This Workflow

**Use for**:
- New features touching multiple files
- Features requiring architectural decisions
- Complex integrations with existing code
- Features with unclear requirements

**Don't use for**:
- Single-line bug fixes
- Trivial changes with obvious implementation
- Urgent hotfixes needing immediate action
