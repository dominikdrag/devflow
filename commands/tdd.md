---
description: Test-Driven Development workflow with Red-Green-Refactor cycles
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite, AskUser
argument-hint: [--explorers=N] [--planners=N] [--architects=N] [--reviewers=N] <feature-description>
---

# TDD Development Workflow

You are guiding the user through a systematic 9-phase Test-Driven Development process. This workflow ensures tests are written BEFORE implementation, following the Red-Green-Refactor cycle.

## Core TDD Principles

1. **Red**: Write a failing test first - define expected behavior before code exists
2. **Green**: Write minimal code to make the test pass - nothing more
3. **Refactor**: Clean up the code while keeping tests green
4. **Tests drive architecture**: Design tests from requirements, then design code to pass them

---

## Workflow State Management

This workflow uses a state file (`claude-tmp/tdd-state.json`) to persist progress across conversation compaction and session interruptions.

### On Workflow Start

**FIRST**, check if `claude-tmp/tdd-state.json` exists:
- **If file exists and `active: true`**: This is a RESUMED workflow
  - Read the state file to understand current progress
  - **If `claude-tmp/tdd-plan.md` exists**: Read the plan file
    - Identify the current task and substep from state file
    - Count remaining unchecked tasks (lines matching `- [ ]`)
    - Display: "Current task: {currentTask.id} ({currentTask.substep}), {N} tasks remaining"
  - Inform the user: "Resuming TDD workflow from Phase {currentPhase}"
  - Display historical context from `phaseHistory`:
    - For each completed phase, show phase name and key outputs
    - Phase 2: Show key files and patterns discovered
    - Phase 3: Show clarifications made
    - Phase 4: Show test cases planned
    - Phase 5: Show selected architecture and rationale
  - Continue from the current phase (do NOT restart from Phase 1)
- **If file does not exist**: This is a NEW workflow
  - Create initial state file:
    ```json
    {
      "active": true,
      "workflowType": "tdd",
      "featureDescription": "[from user input]",
      "startedAt": "[current ISO timestamp]",
      "lastUpdatedAt": "[current ISO timestamp]",
      "currentPhase": 1,
      "currentTask": null,
      "phaseHistory": [],
      "decisions": {"testPlan": null, "architecture": null},
      "summary": "Starting TDD development workflow"
    }
    ```
  - Proceed with Phase 1

### State File Format

```json
{
  "active": true,
  "workflowType": "tdd",
  "featureDescription": "...",
  "startedAt": "ISO timestamp",
  "lastUpdatedAt": "ISO timestamp",
  "currentPhase": 1,
  "currentTask": {
    "id": "TDD-001",
    "substep": "red|green|refactor",
    "attempts": 0,
    "errors": []
  },
  "phaseHistory": [
    {
      "phase": 4,
      "name": "Test Planning",
      "status": "completed",
      "startedAt": "ISO timestamp",
      "completedAt": "ISO timestamp",
      "outputs": {
        "agentCount": 1,
        "testCases": ["test case 1", "test case 2"],
        "testPlanApproved": true
      }
    }
  ],
  "decisions": {
    "testPlan": null,
    "architecture": null
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
| 4 | Test Planning | `agentCount`, `testCases[]`, `testPlanApproved` |
| 5 | Architecture Design | `agentCount`, `optionsPresented[]`, `selectedArchitecture`, `selectionRationale` |
| 6 | Planning | `tddTaskCount`, `planFile` |
| 7 | TDD Implementation | `completedTasks[]`, `blockedTasks[]`, `skippedRefactors[]` |
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
- Store any decisions made (test plan approval, architecture selection) in `decisions`

---

## Configuration

### Parse Arguments

Arguments: $ARGUMENTS

Parse optional flags to configure agent counts:
- `--explorers=N` - Number of `code-explorer` agents (default: 3)
- `--planners=N` - Number of `tdd-test-planner` agents (default: 1)
- `--architects=N` - Number of `code-architect` agents (default: 3)
- `--reviewers=N` - Number of `code-reviewer` agents (default: 3)

Valid range: 1-10 for explorers, 1-3 for planners, 1-5 for architects and reviewers. If a value is out of range, use the closest value in range.

Remaining text after flags is the feature description.

### Display Configuration

At the start, confirm the configuration:
> Using agent counts: {explorers} explorers, {planners} planners, {architects} architects, {reviewers} reviewers

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

## Phase 4: Test Planning

**Goal**: Design tests from requirements BEFORE any code exists

This is the core TDD differentiator - tests are designed from requirements, not from code.

**Actions**:
1. Launch {planners} `tdd-test-planner` agent(s):
   - **If 1**: Single comprehensive test design covering all requirements
   - **If 2+**: Distribute across core functionality, edge cases, and integration tests
2. **WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned
3. Present the FULL test design output to the user - do NOT summarize or condense
4. Use `AskUserQuestion` with options:
   - "Proceed with this test plan"
   - "Modify test plan" (user describes changes)
   - "Add more test cases"
5. If user selects modification:
   - Wait for user input
   - Update the test plan accordingly
   - Re-present and ask again
6. Store approved test plan in state file: `decisions.testPlan`

**CRITICAL**: Do NOT proceed to Phase 5 until user has explicitly approved the test plan via `AskUserQuestion`. The response IS the approval gate.

**IMPORTANT**: Present the FULL output from tdd-test-planner agent(s) to the user - do NOT summarize or condense. The user needs complete visibility into each proposed test case, its rationale, and expected behaviors to make an informed decision. This is a key decision point requiring maximum user control.

**Output**: User-approved test specifications that will drive implementation

---

## Phase 5: Architecture Design

**Goal**: Design the implementation approach to pass the approved tests

**Actions**:
1. Launch {architects} `code-architect` agent(s) with different focuses:
   - **If 1**: Single comprehensive architecture covering all aspects
   - **If 2**: Split between (1) core implementation and (2) integration/edge cases
   - **If 3+**: Distribute across core implementation, integration, and edge cases/error handling
2. **Important context**: Share the approved test plan with architects so they design code that will pass those tests
3. When agents complete, analyze their proposals for similarity:
   - Compare: file structures, component breakdowns, data flows, technology choices
   - **If proposals diverge significantly**: Present as DISTINCT architecture options
   - **If proposals converge with minor variations**: Present the converged approach AND highlight variations as sub-options
   - **If proposals fully converge**: Present unified approach with note that agents agreed
4. Present architecture(s) to the user:
   - **For divergent options**: Present each with name, key decisions, files, pros/cons
   - **For converged with variations**: Present unified core approach, then list each variation point
   - **For fully converged**: Present unified approach
5. Use `AskUserQuestion` tool to get EXPLICIT selection:
   - **For divergent options**: Offer each architecture as a numbered option
   - **For converged with variations**: Present the converged approach, then ask about variation points
   - **ALWAYS** include "Custom: I'll describe my own approach" as final option
6. If user selects custom approach:
   - Wait for user to describe their approach
   - Summarize and confirm with another `AskUserQuestion`
7. Document the selected architecture before proceeding

**CRITICAL**: Do NOT proceed to Phase 6 until user has made an explicit selection via `AskUserQuestion`. The response IS the approval gate.

**IMPORTANT**: Present the FULL output from architecture agents to the user - do NOT summarize or condense their proposals. The user needs complete visibility into each architect's reasoning, trade-offs, and implementation details to make an informed decision.

**Output**: User-selected architecture blueprint designed to pass the approved tests

---

## Phase 6: Planning

**Goal**: Create a comprehensive TDD implementation plan

**Actions**:
1. **Consolidate Information**: Gather all outputs from Phases 1-5:
   - Feature requirements and constraints (Discovery)
   - Codebase patterns and integration points (Exploration)
   - Clarification decisions (Q&A)
   - Approved test plan (Test Planning)
   - Selected architecture with rationale (Architecture Design)

2. **Define TDD Tasks**: Break down the work into discrete tasks, each with Red-Green-Refactor substeps:
   - Create tasks for each testable unit of work
   - Each task should have:
     - Test specification (from Phase 4)
     - Implementation target (from Phase 5)
     - Substeps: TDD-NNN-RED, TDD-NNN-GREEN, TDD-NNN-REFACTOR
   - Establish task dependencies where needed

3. **Define Acceptance Criteria**: Extract or derive acceptance criteria from requirements
   - Each criterion should be testable/verifiable
   - Assign IDs: `AC-NNN`

4. **Write Plan File**: Create `claude-tmp/tdd-plan.md` with this structure:
   ```markdown
   # TDD Development Plan

   > **Status**: In Progress
   > **Methodology**: Test-Driven Development (Red-Green-Refactor)
   > **Current Task**: [none]
   > **Current Substep**: [none]
   > **Created**: [ISO timestamp]
   > **Last Updated**: [ISO timestamp]

   ## Feature Overview
   [Summary from Phase 1]

   ## Codebase Context
   [Key patterns and integration points from Phase 2]

   ## Clarifications
   [Key decisions from Phase 3]

   ## Approved Test Plan
   [Test specifications from Phase 4]

   ## Selected Architecture
   [Chosen architecture and rationale from Phase 5]

   ## TDD Tasks

   Each task follows the Red-Green-Refactor cycle:
   - **Red**: Write failing test that defines expected behavior
   - **Green**: Write minimal code to make the test pass
   - **Refactor**: (Optional) Clean up code while keeping tests green

   ### TDD-001: [Task description]
   - **Test file**: `path/to/test.test.ts`
   - **Implementation files**: `path/to/impl.ts`
   - **Depends on**: none
   - [ ] **TDD-001-RED**: Write failing tests
   - [ ] **TDD-001-GREEN**: Implement to pass
   - [ ] **TDD-001-REFACTOR**: (Optional) Cleanup

   ### TDD-002: [Task description]
   - **Test file**: `path/to/test.test.ts`
   - **Implementation files**: `path/to/impl.ts`
   - **Depends on**: TDD-001
   - [ ] **TDD-002-RED**: Write failing tests
   - [ ] **TDD-002-GREEN**: Implement to pass
   - [ ] **TDD-002-REFACTOR**: (Optional) Cleanup

   ## Quality Tasks
   - [ ] **REVIEW-001**: Run code reviewers on completed implementation
   - [ ] **REVIEW-002**: Apply selected fixes

   ## Acceptance Criteria
   - [ ] **AC-001**: [criterion]
   - [ ] **AC-002**: [criterion]

   ## Progress Log
   | Timestamp | Task | Substep | Event |
   |-----------|------|---------|-------|
   | [ISO] | - | - | Planning phase completed |
   ```

5. **Present Full Plan**: Display the complete plan content to the user by reading and showing `claude-tmp/tdd-plan.md`

**IMPORTANT**: Present the FULL plan to the user - do NOT summarize or condense. The user needs complete visibility into every task, its test specifications, and acceptance criteria to make an informed approval decision.

6. **Plan Approval**: Use `AskUserQuestion` with options:
   - "Proceed with this plan"
   - "Modify the plan" (user describes changes)
   - "Add more tasks"

7. If user selects "Modify the plan" or "Add more tasks":
   - Wait for user input
   - Update the plan file accordingly
   - Re-present summary and ask again

8. **Finalize**: Add approval timestamp to progress log

**CRITICAL**: Do NOT proceed to Phase 7 until user explicitly approves the plan via `AskUserQuestion`.

**Output**: `claude-tmp/tdd-plan.md` file ready to guide TDD implementation

---

## Phase 7: TDD Implementation

**Goal**: Build the feature using Red-Green-Refactor for each task

**CRITICAL GATES** (verify before ANY implementation):
- [ ] Test plan approved via `AskUserQuestion` in Phase 4
- [ ] Architecture selected via `AskUserQuestion` in Phase 5
- [ ] Plan approved via `AskUserQuestion` in Phase 6

If any gate is missing, STOP and complete the required phase first.

**For Each TDD-NNN Task**:

### Step 7.1: RED - Write Failing Test

1. Update state: `currentTask: { id: "TDD-NNN", substep: "red", attempts: 0 }`
2. Read the test specification from the approved test plan
3. Write the test file following project conventions
4. Launch `test-runner` agent to verify test FAILS
5. **If test passes** (should not happen in RED):
   - This means the test is not testing new behavior
   - Inform user: "Test passed but should fail in RED phase - test may not be verifying new behavior"
   - Use `AskUserQuestion`:
     - "Revise the test to properly fail"
     - "Continue anyway (test already passes)"
   - If revise: update test and re-run
6. When test fails correctly:
   - Update plan: Mark `[x] TDD-NNN-RED`, add `Completed: [timestamp]`
   - Add progress log entry: `| [timestamp] | TDD-NNN | RED | Tests written (failing) |`

### Step 7.2: GREEN - Implement Minimal Code

1. Update state: `currentTask.substep: "green"`, `attempts: 0`
2. Write minimal implementation to make test pass - nothing more
3. Launch `test-runner` agent to verify test PASSES
4. **If test fails**:
   - Increment `state.currentTask.attempts`
   - Log error to `state.currentTask.errors[]`
   - **If attempts <= 3**: Analyze failure, fix implementation, retry
   - **If attempts > 3**: Mark task as BLOCKED
     - Use `AskUserQuestion`:
       - "Debug manually and retry" - Pause for user investigation
       - "Skip this task and continue" - Mark incomplete, proceed
       - "Abort TDD workflow" - Clean termination
       - "Modify the test" - Return to RED (breaks TDD discipline)
     - Log: `| [timestamp] | TDD-NNN | GREEN | BLOCKED after 3 attempts |`
5. When test passes:
   - Update plan: Mark `[x] TDD-NNN-GREEN`, add `Completed: [timestamp]`
   - Add progress log entry: `| [timestamp] | TDD-NNN | GREEN | Implementation complete |`

### Step 7.3: REFACTOR (Optional)

1. Update state: `currentTask.substep: "refactor"`
2. Use `AskUserQuestion`:
   - "Refactor this implementation" - Apply cleanup
   - "Skip refactoring, continue to next task" - Mark complete, move on
   - "Review code before deciding" - Show implementation for review
3. **If user chooses refactor**:
   - Apply improvements (no behavior change - tests must stay green)
   - Launch `test-runner` to run ALL TDD tests (not just current task)
   - **If any test fails** (regression):
     - Inform user which test failed
     - Use `AskUserQuestion`:
       - "Revert refactoring" - Undo changes, mark skipped
       - "Fix the regression" - Attempt fix (max 2 attempts)
     - If regression persists after 2 attempts, force revert
   - When all tests pass:
     - Update plan: Mark `[x] TDD-NNN-REFACTOR`, add `Completed: [timestamp]`
     - Add progress log entry
4. **If user skips**:
   - Update plan: Mark `[SKIPPED] TDD-NNN-REFACTOR`
   - Add progress log entry: `| [timestamp] | TDD-NNN | REFACTOR | Skipped by user |`

### Step 7.4: Proceed to Next Task

1. Add task to `completedTasks[]` in state
2. Clear `currentTask` or set to next task
3. If all tasks complete, proceed to Phase 8
4. Otherwise, continue with next TDD-NNN task

**Guidelines**:
- Follow existing code patterns exactly
- Write MINIMAL code to pass tests - no gold plating
- Don't add unrequested features
- Don't refactor unrelated code
- Update plan file after each substep completion

**Output**: Implemented feature with comprehensive test coverage, all tests passing

---

## Phase 8: Quality Review

**Goal**: Ensure code quality and correctness through user-reviewed findings

**Scope**: This phase runs AFTER all TDD cycles are complete (not per-task).

**Actions**:

### Step 1: Review Existing Review Tasks
1. Read `claude-tmp/tdd-plan.md` and extract all `REVIEW-NNN` tasks
2. Note any blocked or skipped TDD tasks (they won't be reviewed)

### Step 2: Launch Review Agents
1. Launch {reviewers} `code-reviewer` agent(s) in parallel:
   - **If 1**: Comprehensive review covering all aspects
   - **If 2**: Split between (1) correctness/bugs and (2) conventions/maintainability
   - **If 3+**: Distribute across correctness, conventions, and maintainability
2. **WAIT for ALL agents to complete** before proceeding
3. **Optional**: If user requests security audit, also launch `security-auditor` agent and wait

### Step 3: Present Full Agent Output
**IMPORTANT**: Present the FULL output from each review agent to the user - do NOT summarize or condense their findings. The user needs complete visibility into each reviewer's analysis, reasoning, and specific concerns to make informed decisions about which issues to address.

### Step 4: Organize Findings
1. Map high-confidence issues (>=80) to actionable review tasks
2. Deduplicate overlapping issues (same file + line + similar description)
3. Organize by severity:
   - **Critical Issues** (Confidence 90-100): Must be fixed
   - **Important Issues** (Confidence 80-89): Should be addressed
   - **Suggestions** (Confidence < 80): Nice to have improvements

### Step 5: Present Findings Summary
Display:
```
## Review Findings Summary

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
Use `AskUserQuestion` with `multiSelect: true` to let user choose which issues to address:
- List each issue as a selectable option
- Group by severity in the question
- Include "Skip all - proceed to summary" as an option

### Step 7: Apply Fixes
For each selected issue:
1. Apply the fix
2. Launch `test-runner` to verify ALL TDD tests still pass
3. If any test fails: revert fix, inform user, skip that fix
4. Mark REVIEW-NNN task complete in plan file

### Step 8: Offer Re-review
If any fixes were applied, use `AskUserQuestion` to ask:
- "Run review again to verify fixes?"
- "Proceed to summary"

If user chooses re-review, return to Step 2 with focused scope.

**Output**: Quality-verified implementation with all tests passing

---

## Phase 9: Summary

**Goal**: Document what was accomplished and clean up workflow state

**Actions**:
1. List all files created or modified
2. Summarize key decisions made:
   - Test plan approach
   - Architecture selection
   - Refactoring decisions
3. Document test coverage achieved:
   - Number of test cases written
   - Types of tests (unit, integration, boundary, failure)
4. Note any deferred work or known limitations:
   - Blocked tasks (if any)
   - Skipped refactoring (if any)
5. Suggest potential follow-up tasks
6. Mark all todos as complete

**Plan File Updates**:
1. Update header: Set `Status: Complete` (or `Status: Incomplete` if any tasks blocked), update `Last Updated`
2. Mark acceptance criteria: Change `- [ ]` to `- [x]` for completed, add notes for incomplete
3. Add final progress log entries:
   - `| [timestamp] | - | - | Acceptance criteria reviewed |`
   - `| [timestamp] | - | - | TDD development completed |`

**Cleanup** (conditional):
- Delete state file (`claude-tmp/tdd-state.json`)
- **Only delete plan file if ALL tasks and acceptance criteria are complete**
- If any tasks are incomplete, keep `tdd-plan.md` as a record of unfinished work

**Output**: Completion summary with test coverage report and next steps

---

## Usage

This workflow is invoked with `/tdd` followed by optional flags and a feature description:

### Basic Usage
```
/tdd Add user authentication with OAuth support
/tdd
```

### With Agent Count Overrides
```
/tdd --explorers=2 --architects=2 Add user authentication
/tdd --planners=2 --reviewers=5 Implement payment processing
/tdd --explorers=1 --architects=1 --reviewers=1 Small utility function
```

### Flag Reference

| Flag | Default | Range | Phase |
|------|---------|-------|-------|
| `--explorers=N` | 3 | 1-10 | Phase 2: Codebase Exploration |
| `--planners=N` | 1 | 1-3 | Phase 4: Test Planning |
| `--architects=N` | 3 | 1-5 | Phase 5: Architecture Design |
| `--reviewers=N` | 3 | 1-5 | Phase 8: Quality Review |

If no description is provided, ask the user what feature they want to build.

## When to Use This Workflow

**Use for**:
- Features where you want tests to drive design decisions
- Complex business logic that benefits from test-first thinking
- Features with well-defined acceptance criteria
- When you want comprehensive test coverage as a natural byproduct

**Don't use for**:
- Exploratory prototyping where requirements are unclear
- Simple UI changes with no testable logic
- Urgent hotfixes where speed trumps test coverage
- Features where the `/feature` workflow is more appropriate

## Key Differences from `/feature`

| Aspect | `/feature` | `/tdd` |
|--------|------------|--------|
| Test timing | After implementation (Phase 7) | Before implementation (Phase 4 + per-task RED) |
| Test agent | `test-analyzer` (analyzes code) | `tdd-test-planner` (designs from requirements) |
| Implementation | Linear per-task | Red-Green-Refactor per-task |
| Refactoring | Not explicit | Explicit optional step |
| Test coverage | Proposed after code | Guaranteed by design |
