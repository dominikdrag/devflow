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
  - Display completed phases and key decisions from the state
  - Continue from the current phase (do NOT restart from Phase 1)
- **If file does not exist**: This is a NEW workflow
  - Create initial state file with `active: true, currentPhase: 1, completedPhases: []`
  - Proceed with Phase 1

### State File Format

```json
{
  "active": true,
  "currentPhase": 1,
  "completedPhases": [],
  "featureDescription": "...",
  "currentTask": null,
  "decisions": {
    "architecture": null,
    "testStrategy": null
  },
  "summary": "Brief context for resumption"
}
```

### Updating State

At the START of each phase, update the state file:
- Set `currentPhase` to the new phase number
- Update `summary` with relevant context

At the END of each phase, update the state file:
- Add the phase number to `completedPhases`
- Store any decisions made (architecture selection, test strategy approval)

---

## Configuration

### Parse Arguments

Arguments: $ARGUMENTS

Parse optional flags to configure agent counts:
- `--explorers=N` - Number of `code-explorer` agents (default: 3)
- `--architects=N` - Number of `code-architect` agents (default: 3)
- `--analyzers=N` - Number of `test-analyzer` agents (default: 1)
- `--reviewers=N` - Number of `code-reviewer` agents (default: 3)

Valid range for all counts: 1-5. If a value is out of range, use the default.

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
   - Group tasks by phase (Implementation, Testing, Quality)
   - Establish task dependencies where needed
   - Assign task IDs: `TASK-NNN` for implementation, `TEST-NNN` for testing, `REVIEW-NNN` for quality

3. **Define Acceptance Criteria**: Extract or derive acceptance criteria from requirements
   - Each criterion should be testable/verifiable
   - Assign IDs: `AC-NNN`

4. **Write Plan File**: Create `claude-tmp/devflow-plan.md` with this structure:
   ```markdown
   # Feature Development Plan

   > **Status**: In Progress
   > **Current Task**: [none]
   > **Created**: [ISO timestamp]
   > **Last Updated**: [ISO timestamp]

   ## Feature Overview
   [Summary from Phase 1]

   ## Codebase Context
   [Key patterns and integration points from Phase 2]

   ## Clarifications
   [Key decisions from Phase 3]

   ## Selected Architecture
   [Chosen architecture and rationale from Phase 4]

   ## Implementation Tasks
   - [ ] **TASK-001**: [description]
     - Files: `path/to/file.ts`
     - Depends on: none
   - [ ] **TASK-002**: [description]
     - Files: `path/to/file.ts`
     - Depends on: TASK-001

   ## Testing Tasks
   - [ ] **TEST-001**: [description]

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

5. **Present Plan Summary**: Display to the user:
   - Total task count by category (Implementation, Testing, Quality)
   - Files to be created/modified
   - Acceptance criteria count

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

**Actions**:
1. **Only begin after explicit user approval of plan in Phase 5**
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

**Approach**: Use a `test-analyzer` agent to propose test cases, present the analysis to the user for approval, then write tests directly (not delegated) to preserve local context from implementation. Use `test-runner` agent to execute tests and report results.

**Actions**:
1. Launch {analyzers} `test-analyzer` agent(s) to analyze the implemented changes:
   - **If 1**: Single comprehensive test analysis
   - **If 2+**: Distribute across unit tests, integration tests, and edge cases
   - Each will identify test patterns, propose test cases, and note mocking requirements
2. Present the test analysis to the user:
   - Summarize the proposed test categories (unit, integration, edge cases)
   - List key test cases with their purposes
   - Highlight any mocking requirements or special setup needed
   - Note the total number of tests proposed
3. Use `AskUserQuestion` to get EXPLICIT confirmation:
   - Option 1: "Proceed with proposed testing strategy"
   - Option 2: "Modify testing scope" (user describes changes)
   - Option 3: "Skip testing phase"
4. If user selects "Modify testing scope":
   - Wait for user to describe their modifications
   - Incorporate feedback into test plan
5. Write tests following the confirmed plan and project conventions:
   - Happy path scenarios
   - Edge cases and boundary conditions
   - Error handling paths
   - Integration points with existing code
6. **Execute tests using `test-runner` agent**:
   - Pass the test files/patterns from the approved test plan
   - Agent runs tests and returns structured results
7. **Handle test results**:
   - If all tests pass, proceed to Phase 8
   - If tests fail:
     - Review the failure report from `test-runner`
     - Fix the failing tests (adjust test code or implementation as needed)
     - Re-run `test-runner` until all pass

**CRITICAL**: Do NOT write tests until user has made an explicit selection via `AskUserQuestion`.

**IMPORTANT**: Present the FULL output from test-analyzer agent(s) to the user - do NOT summarize or condense the test proposals. The user needs complete visibility into each proposed test case, its rationale, edge cases identified, and mocking requirements to make an informed decision about the testing strategy.

**Why this approach**:
- Analyzer agent provides structured, comprehensive test proposals
- User approval ensures alignment with expectations before effort is spent
- Writing tests directly preserves local context from implementation
- Test-runner agent provides structured, parseable test results
- Results in better test quality than delegating to an agent without context

**Test Quality Standards**:
- Test names clearly describe what is being tested
- Each test focuses on a single behavior
- Tests are independent (no shared state)
- Follow existing project test patterns exactly

**Output**: User-approved test suite with passing tests

---

## Phase 8: Quality Review

**Goal**: Ensure code quality and correctness through user-reviewed findings

**Actions**:

### Step 1: Launch Review Agents
1. Launch {reviewers} `code-reviewer` agent(s) in parallel:
   - **If 1**: Comprehensive review covering all aspects
   - **If 2**: Split between (1) correctness/bugs and (2) conventions/maintainability
   - **If 3+**: Distribute across correctness, conventions, and maintainability
2. **Wait for ALL agents to complete** before proceeding
3. **Optional**: If user requests security audit, also launch `security-auditor` agent and wait

### Step 2: Present Full Agent Output
**IMPORTANT**: Present the FULL output from each review agent to the user - do NOT summarize or condense their findings. The user needs complete visibility into each reviewer's analysis, reasoning, and specific concerns to make informed decisions about which issues to address. This is a critical quality gate requiring maximum user control.

### Step 3: Consolidate Findings
1. Collect all findings from completed agents
2. Deduplicate overlapping issues (same file + line + similar description)
3. Organize by severity:
   - **Critical Issues** (Confidence 90-100): Must be fixed
   - **Important Issues** (Confidence 80-89): Should be addressed
   - **Suggestions** (Confidence < 80): Nice to have improvements

### Step 4: Present Findings to User
Display consolidated findings in a clear format:

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

### Step 5: User Selection
Use `AskUserQuestion` with `multiSelect: true` to let user choose which issues to address:
- List each issue as a selectable option
- Group by severity in the question
- Include "Skip all - proceed to summary" as an option

### Step 6: Apply Selected Fixes
1. Apply fixes ONLY for issues the user selected
2. Track which fixes were applied for the summary

### Step 7: Offer Re-review
If any fixes were applied, use `AskUserQuestion` to ask:
- "Run review again to verify fixes?"
- "Proceed to summary"

If user chooses re-review, return to Step 1 with a focused scope.

**Output**: Quality-verified implementation with user-approved fixes

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
7. **Mark all acceptance criteria** in plan file as complete (or note incomplete ones)
8. **Delete the state file** (`claude-tmp/devflow-state.json`) to mark workflow as complete
9. **Delete the plan file** (`claude-tmp/devflow-plan.md`) to clean up

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
| `--explorers=N` | 3 | 1-5 | Phase 2: Codebase Exploration |
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
