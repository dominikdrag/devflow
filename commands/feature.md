---
description: Guided feature development with codebase understanding and architecture focus
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite, AskUser
argument-hint: <feature-description>
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

This workflow uses a state file (`claude-tmp/devflow-feature-state.json`) to persist progress across conversation compaction and session interruptions.

### On Workflow Start

**FIRST**, check if `claude-tmp/devflow-feature-state.json` exists:
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
| 2 | Codebase Exploration | `focusesSelected[]`, `keyFiles[]`, `patterns[]`, `integrationPoints[]` |
| 3 | Clarifying Questions | `clarifications[]` (array of `{question, answer}`) |
| 4 | Architecture Design | `perspectivesSelected[]`, `optionsPresented[]`, `selectedArchitecture`, `selectionRationale` |
| 5 | Planning | `taskCount`, `testCount`, `reviewCount`, `planFile` |
| 6 | Implementation | `tasksCompleted[]`, `tasksRemaining[]`, `filesModified[]` |
| 7 | Testing | `focusesSelected[]`, `testsWritten[]`, `testsPassing` |
| 8 | Quality Review | `focusesSelected[]`, `issuesFound`, `issuesFixed[]`, `issuesSkipped[]` |
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

The entire argument text is the feature description. All phases with agent selection are now interactive.

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

**Goal**: Understand existing patterns and relevant code through focused exploration

**Actions**:

### Step 1: Present Exploration Focus Options

Display available exploration focuses for user selection:

```
## Codebase Exploration

Select which focuses to explore. Each launches a parallel explorer agent.

  Core Focuses (recommended for most features)
   1. Similar Features      - Existing features to use as templates, reusable patterns
   2. Integration Points    - Where feature connects to existing systems, APIs, modules
   3. Data Flow & State     - How data moves, storage interactions, state management

  Specialized Focuses (use when relevant)
   4. Entry Points          - API surfaces, UI components, CLI commands, event handlers
   5. Patterns & Conventions - Design patterns, coding conventions, project structure
   6. Error Handling        - Error paths, validation, edge cases, failure modes

Enter selection (e.g., "1,2,3" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Step 2: Parse Selection

Accept input formats:
- Comma-separated: `1,2,3`
- Ranges: `1-3`
- Mixed: `1-3,5`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-6 only, deduplicate, sort.

### Step 3: Launch Selected Explorers

Launch one `code-explorer` agent per selected focus, all in parallel.

**Agent Prompt Format**: Include the focus assignment in each agent's prompt:
> Your assigned exploration focus is: **[Focus Name]**
>
> [Include the FULL focus definition from the reference below]
>
> Explore the codebase for: [feature description]
>
> Requirements context: [key requirements from Phase 1]

**Focus Definitions** (inject the selected one into each agent prompt):

1. **Similar Features**
   - Focus: Find existing features to use as implementation templates
   - When to use: Most features - provides patterns and prior art
   - Search for features with similar functionality or UX patterns
   - Identify reusable components, utilities, and abstractions
   - Document how similar features handle edge cases
   - Note file organization and naming conventions used

2. **Integration Points**
   - Focus: Map where the feature connects to existing systems
   - When to use: Features that interact with existing modules, APIs, or services
   - Identify modules/services the feature will call or depend on
   - Find existing APIs and interfaces to integrate with
   - Document authentication, authorization, and data access patterns
   - Note configuration and dependency injection patterns

3. **Data Flow & State**
   - Focus: Understand how data moves and is stored
   - When to use: Features involving data persistence, caching, or state management
   - Trace data from entry to storage and back
   - Map database schemas, models, and repositories
   - Identify caching layers and invalidation patterns
   - Document state management approaches (stores, contexts, etc.)

4. **Entry Points**
   - Focus: Find where users/systems will trigger this feature
   - When to use: Features with multiple entry points or complex routing
   - Locate API endpoint definitions and routing
   - Find UI component mounting and navigation
   - Identify CLI command registration
   - Map event handlers and message consumers

5. **Patterns & Conventions**
   - Focus: Understand project structure and coding standards
   - When to use: Unfamiliar codebases or when conventions are unclear
   - Read CLAUDE.md, style guides, and contributing docs
   - Identify folder structure and file naming conventions
   - Document common patterns (error handling, logging, testing)
   - Note linting rules and code formatting standards

6. **Error Handling**
   - Focus: Map error paths and failure modes
   - When to use: Critical features, payment flows, data mutations
   - Identify error handling patterns in similar features
   - Document validation approaches and error messages
   - Find retry logic, fallbacks, and recovery patterns
   - Note logging and monitoring for errors

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Synthesize Findings

1. Read the essential files identified by all agents
2. Combine findings into a unified codebase understanding
3. Note any conflicts or inconsistencies between agent findings
4. Prepare context summary for subsequent phases

**CRITICAL**: You MUST wait for ALL exploration agents to return their complete output before proceeding to Phase 3. The exploration results are essential for asking informed clarifying questions. NEVER proceed to Phase 3 while agents are still running or before reviewing their findings.

**Output**: Synthesized summary of patterns, conventions, and integration points from all exploration focuses

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

**Goal**: Design the implementation approach with user-selected perspectives

**Actions**:

### Step 1: Present Perspective Options

Display available architecture perspectives for user selection:

```
## Architecture Design

Select which perspectives to explore. Each launches a parallel architect agent.

  Core Approaches (general-purpose)
   1. Minimal Changes      - Smallest change, maximum reuse
   2. Clean Architecture   - Elegant abstractions, separation of concerns
   3. Pragmatic Balance    - Trade-off between speed and quality

  Specialized Approaches (use when applicable)
   4. Performance-First    - For: hot paths, high-throughput, latency-sensitive
   5. Security-First       - For: auth, payments, PII, external APIs
   6. Scalability-First    - For: >1000 concurrent users, distributed state
   7. Testability-First    - For: complex business logic, financial calculations
   8. Migration-Focused    - For: replacing existing features, gradual rollout
   9. Domain-Driven        - For: complex domains, multiple subdomains
  10. Event-Driven         - For: workflows, notifications, audit trails

Enter selection (e.g., "1,3,5" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Step 2: Parse Selection

Accept input formats:
- Comma-separated: `1,3,5`
- Ranges: `1-3`
- Mixed: `1-3,5,7`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-10 only, deduplicate, sort.

### Step 3: Launch Selected Architects

Launch one `code-architect` agent per selected perspective, all in parallel.

**Agent Prompt Format**: Include the perspective assignment in each agent's prompt:
> Your assigned perspective is: **[Perspective Name]**
>
> [Include the FULL perspective definition from the reference below]
>
> Design an architecture for: [feature description]
>
> Context from exploration: [key findings from Phase 2]
>
> Requirements clarified: [key clarifications from Phase 3]

**Perspective Definitions** (inject the selected one into each agent prompt):

1. **Minimal Changes**
   - Philosophy: Smallest change, maximum reuse of existing code
   - Extend existing components rather than creating new ones
   - Reuse current patterns and abstractions
   - Minimize file count and refactoring scope
   - Prioritize speed and low risk over architectural purity

2. **Clean Architecture**
   - Philosophy: Maintainability through elegant abstractions and separation of concerns
   - Create dedicated services/modules with clear interfaces
   - Apply separation of concerns rigorously
   - Use dependency injection and clear boundaries
   - Prioritize testability and long-term maintainability over speed

3. **Pragmatic Balance**
   - Philosophy: Balance between speed and quality - good boundaries without excessive overhead
   - Create focused abstractions only where they add clear value
   - Integrate with existing code through composition
   - Apply clean architecture principles selectively
   - Prioritize reasonable trade-offs between speed and maintainability

4. **Performance-First**
   - Philosophy: Optimize for speed and resource efficiency from the start
   - When to use: Hot paths, high-throughput APIs, latency-sensitive operations, resource-constrained environments
   - Profile-driven design decisions
   - Minimize allocations, copies, and indirection
   - Choose data structures for access patterns
   - Consider cache locality and memory layout
   - Prioritize hot path optimization over code elegance

5. **Security-First**
   - Philosophy: Threat modeling and principle of least privilege built into design
   - When to use: Authentication, authorization, payment processing, PII handling, external API integrations
   - Defense in depth at every layer
   - Input validation at all boundaries
   - Principle of least privilege for all components
   - Secure defaults, explicit opt-in for dangerous operations
   - Audit logging and traceability built-in

6. **Scalability-First**
   - Philosophy: Design for horizontal scaling and distributed systems
   - When to use: Features expecting >1000 concurrent users, distributed state, multi-region deployments
   - Stateless components where possible
   - Eventual consistency where appropriate
   - Design for partition tolerance
   - Consider queue-based decoupling
   - Plan for data sharding and replication

7. **Testability-First**
   - Philosophy: Design for easy testing through pure functions and dependency injection
   - When to use: Complex business logic, financial calculations, rule engines, code requiring high correctness guarantees
   - Pure functions with explicit inputs/outputs
   - Dependency injection for all external services
   - Clear seams for mocking and stubbing
   - Separate side effects from business logic
   - Design for deterministic, fast tests

8. **Migration-Focused**
   - Philosophy: Design for gradual adoption and rollout
   - When to use: Replacing existing functionality, risky changes to critical paths, features needing A/B testing or gradual rollout
   - Feature flags for incremental rollout
   - Backwards compatibility with existing interfaces
   - Parallel run capability for validation
   - Fallback mechanisms for safe rollback
   - Minimize blast radius of changes

9. **Domain-Driven**
   - Philosophy: Organize around business concepts and bounded contexts
   - When to use: Complex business domains, features spanning multiple subdomains, systems requiring ubiquitous language alignment
   - Identify and respect bounded contexts
   - Use ubiquitous language from the domain
   - Aggregate roots for consistency boundaries
   - Domain events for cross-context communication
   - Rich domain models over anemic data structures

10. **Event-Driven**
    - Philosophy: Async patterns, messaging, and loose coupling
    - When to use: Workflows, notifications, audit trails, integrations with external systems, features requiring loose coupling
    - Events as first-class citizens
    - Publish-subscribe for loose coupling
    - Event sourcing where audit trails matter
    - Saga patterns for distributed transactions
    - Design for eventual consistency and idempotency

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Present All Proposals

When agents complete, present ALL architecture proposals using this format:

```
### Architecture Options

#### Approach 1: [Perspective Name]
**Philosophy**: [Perspective philosophy]

**Architecture Diagram**:
```
[ASCII diagram from agent output showing components and dependencies]
```

**Key Decisions**:
- [Decision 1]
- [Decision 2]
- [Decision 3]

**Files to Create/Modify**:
- `path/to/file.ts` - [purpose]
- `path/to/file2.ts` - [purpose]

**Pros**:
- [Pro 1]
- [Pro 2]
- [Pro 3]

**Cons**:
- [Con 1]
- [Con 2]
- [Con 3]

---

[Repeat for each selected perspective]

---

### Recommendation

**Approach [N]: [Name]** - [Brief rationale for why this is recommended given the specific context]
```

**IMPORTANT**: Present the FULL output from architecture agents to the user - do NOT summarize or condense their proposals. The user needs complete visibility into each architect's reasoning, trade-offs, and implementation details to make an informed decision. This is a key decision point requiring maximum user control.

### Step 5: User Selection

Ask the user to select an architecture by typing their choice:

```
Which architecture do you want to use?

Enter a number (1-N), or type "custom" to describe your own approach:
```

Wait for user response. Parse their input:
- If a number: select that architecture
- If "custom": wait for user to describe their approach, then summarize and confirm

Document the selected architecture before proceeding.

**CRITICAL**: Do NOT proceed to Phase 5 until user has typed their selection. The response IS the approval gate.

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
   - Group tasks by phase (Implementation, Testing, Review)
   - Establish task dependencies where needed
   - Assign task IDs: `TASK-NNN` for implementation
   - **Note**: `TEST-NNN` and `REVIEW-NNN` tasks are created dynamically in Phases 7 and 8

   **Phase Scope Rules** (document explicitly in plan):
   - **Phase 6 (Implementation)**: Works ONLY on `TASK-NNN` tasks
   - **Phase 7 (Testing)**: Creates `TEST-NNN` tasks from test-analyzer output, then executes them
   - **Phase 8 (Review)**: Creates/refines `REVIEW-NNN` tasks from code-reviewer, then executes them

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
   [Brief description of each architecture option presented, with perspective name]

   ### Selected Architecture: [Perspective Name]

   **Philosophy**: [How the perspective shapes this design]

   **Architecture Diagram**:
   ```
   [ASCII diagram from the selected architecture showing NEW and EXISTING components]
   ```

   **Discovered Patterns**:
   - [Pattern from CLAUDE.md/style guides with file:line reference]
   - [Existing convention with code reference]

   **Architecture Decision**: [The approach with clear rationale]

   **Component Design**:
   | Component | Responsibility | Dependencies | Interface |
   |-----------|----------------|--------------|-----------|
   | [Name] | [What it does] | [What it needs] | [Public API] |

   **Data Flow**: [How information moves through the system]

   **Build Sequence**:
   1. [Phase 1 - what to build first and why]
   2. [Phase 2 - what comes next]

   **Trade-offs**:
   - **Pros**: [advantages of this approach]
   - **Cons**: [disadvantages/risks]

   **Critical Considerations**:
   - **Error Handling**: [approach]
   - **State Management**: [approach]
   - **Testing Strategy**: [approach]
   - **Performance**: [considerations]
   - **Security**: [considerations]

   **Files to Create/Modify**:
   - `path/to/new/file.ts` - [purpose and what it will contain]
   - `path/to/existing/file.ts` - [what specific changes will be made]

   ## Phase Scope Rules
   - **Phase 6 (Implementation)**: Works ONLY on `TASK-NNN` tasks
   - **Phase 7 (Testing)**: Creates `TEST-NNN` tasks from test-analyzer, then executes them
   - **Phase 8 (Review)**: Creates/refines `REVIEW-NNN` tasks from code-reviewer, then executes them

   ## Implementation Tasks
   - [ ] **TASK-001**: [description]
     - Files: `path/to/file.ts`
     - Depends on: none
   - [ ] **TASK-002**: [description]
     - Files: `path/to/file.ts`
     - Depends on: TASK-001

   ## Test Tasks
   > Populated during Phase 7 (Testing) based on test-analyzer agent output and user selection.

   (none yet)

   ## Review Tasks
   > Populated during Phase 8 (Quality Review) based on code-reviewer agent findings and user selection.

   (none yet)

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

6. **Plan Approval**: Ask the user to approve the plan:

```
How would you like to proceed?

Type: "proceed" to approve, "modify" to request changes, or "add" to add more tasks:
```

7. If user types "modify" or "add":
   - Wait for user to describe changes
   - Update the plan file accordingly
   - Re-present plan and ask again

8. **Finalize**: Add approval timestamp to progress log

**CRITICAL**: Do NOT proceed to Phase 6 until user types "proceed".

**Output**: `claude-tmp/devflow-plan.md` file ready to guide implementation

---

## Phase 6: Implementation

**Goal**: Build the feature following the approved plan

**Scope**: This phase works ONLY on `TASK-NNN` implementation tasks. Testing tasks (`TEST-NNN`) and review tasks (`REVIEW-NNN`) are handled in their respective phases.

**CRITICAL GATES** (verify before ANY implementation):
- [ ] Architecture selected in Phase 4 (user typed their choice)
- [ ] Plan approved in Phase 5 (user typed "proceed")

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

**Approach**: Launch test-analyzer agent(s) to propose test cases based on the implemented code. Present proposals to user for approval. **Once user approves the testing strategy, add `TEST-NNN` tasks to `claude-tmp/devflow-plan.md`**. Then write tests directly (not delegated) to preserve local context from implementation.

**Actions**:

### Step 1: Present Test Focus Options

Display available test focuses for user selection:

```
## Test Planning

Select which test focuses to explore. Each launches a parallel test analyzer agent.

  Core Focuses (recommended for most features)
   1. Happy Path          - Normal usage, expected inputs/outputs, core functionality
   2. Edge Cases          - Boundaries, empty/null, max values, special characters
   3. Error Handling      - Invalid inputs, exceptions, failure responses

  Specialized Focuses (use when relevant)
   4. Integration         - For: external services, database, API interactions
   5. State & Mutations   - For: state management, data transformations, side effects
   6. Security            - For: auth, permissions, user input, sensitive data

Enter selection (e.g., "1,2,3" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Step 2: Parse Selection

Accept input formats:
- Comma-separated: `1,2,3`
- Ranges: `1-3`
- Mixed: `1-3,5`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-6 only, deduplicate, sort.

### Step 3: Launch Selected Test Analyzers

Launch one `test-analyzer` agent per selected focus, all in parallel.

**Agent Prompt Format**: Include the focus assignment in each agent's prompt:
> Your assigned test focus is: **[Focus Name]**
>
> [Include the FULL focus definition from the reference below]
>
> Analyze tests needed for: [feature description]
>
> Implementation context: [files modified in Phase 6]

**Focus Definitions** (inject the selected one into each agent prompt):

1. **Happy Path**
   - Focus: Normal usage scenarios that verify core functionality
   - When to use: Always - validates feature works as intended
   - Analyze code for expected inputs producing expected outputs
   - Cover the main success scenarios users will encounter
   - Each test verifies ONE specific behavior
   - Name tests to describe the behavior being verified

2. **Edge Cases**
   - Focus: Boundary conditions and unusual but valid inputs
   - When to use: Always - catches common bugs at boundaries
   - Empty inputs, zero values, maximum values
   - Null/undefined handling where applicable
   - Boundary conditions at limits (off-by-one, etc.)
   - Special characters, unicode, format edge cases

3. **Error Handling**
   - Focus: Invalid inputs and failure responses
   - When to use: Always - ensures graceful failure behavior
   - Invalid input types and formats
   - Missing required data
   - Out-of-range values
   - Expected error messages and response codes

4. **Integration**
   - Focus: Interactions with external systems and components
   - When to use: Features with database, API, or service dependencies
   - Database operations (CRUD, transactions)
   - External API calls (success and failure scenarios)
   - Component interactions across boundaries
   - Mock specifications for external dependencies

5. **State & Mutations**
   - Focus: State changes and data transformations
   - When to use: Features managing state or modifying data
   - State transitions and lifecycle
   - Data transformations at each step
   - Side effects and their verification
   - Rollback and cleanup scenarios

6. **Security**
   - Focus: Authentication, authorization, and input safety
   - When to use: Auth flows, user input handling, sensitive data
   - Authentication failures and token handling
   - Authorization checks and permission boundaries
   - Input sanitization and injection prevention
   - Sensitive data handling

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Present and Approve Test Plan

1. Present the FULL output from all test-analyzer agents - do NOT summarize

2. Ask the user to approve the test plan:

```
How would you like to proceed with testing?

Type: "proceed" to approve, "modify" to change scope, or "skip" to skip testing:
```

**IMPORTANT**: Present the FULL output from all test-analyzer agents to the user - do NOT summarize or condense. The user needs complete visibility into each proposed test case to make an informed decision.

### Step 5: Create TEST Tasks in Plan File

**IMPORTANT**: This is when `TEST-NNN` tasks are created and added to the plan file - NOT during Phase 5 planning. This ensures test proposals are based on actual implementation context.

If user approves the testing strategy:
1. Update `claude-tmp/devflow-plan.md`:
   - Populate the "## Test Tasks" section (already exists with placeholder "(none yet)")
   - Create `TEST-NNN` tasks based on approved test proposals (starting from TEST-001)
   - Each task should specify: test file path, what behavior is tested, edge cases covered
2. Add progress log entry: `| [timestamp] | Testing tasks created from test-analyzer output |`

### Step 6: Execute Testing Tasks
For each TEST-NNN task in the updated plan:
1. Update state file with `currentTask: "TEST-NNN"`
2. Write tests following the task description and project conventions
3. Mark task complete: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]`
4. Add progress log entry: `| [timestamp] | TEST-NNN completed |`

### Step 7: Run Tests
1. Launch `test-runner` agent to execute all new tests
2. If tests fail:
   - Review the failure report
   - Fix the failing tests (adjust test code or implementation as needed)
   - Re-run until all pass
3. When all pass: Add `| [timestamp] | Testing phase completed |`

**CRITICAL**: Do NOT write tests until user has typed "proceed" to approve the testing strategy.

**Test Quality Standards**:
- Test names clearly describe what is being tested
- Each test focuses on a single behavior
- Tests are independent (no shared state)
- Follow existing project test patterns exactly

**Output**: User-approved test suite with passing tests, updated plan file with TEST-NNN tasks

---

## Phase 8: Quality Review

**Goal**: Ensure code quality and correctness through user-selected review focuses

**Scope**: This phase works ONLY on `REVIEW-NNN` quality review tasks. Implementation tasks (`TASK-NNN`) and testing tasks (`TEST-NNN`) were completed in previous phases.

**Actions**:

### Step 1: Present Review Focus Options

Display available review focuses for user selection:

```
## Quality Review

Select which review focuses to explore. Each launches a parallel reviewer agent.

  Core Focuses (recommended for most features)
   1. Correctness & Bugs   - Logic errors, null handling, race conditions
   2. Conventions & Style  - Project guidelines, naming, patterns
   3. Error Handling       - Missing catches, recovery, error messages

  Specialized Focuses (use when relevant)
   4. Security             - For: auth, user input, external APIs, sensitive data
   5. Performance          - For: hot paths, data processing, resource management
   6. Maintainability      - For: large changes, shared code, public APIs

Enter selection (e.g., "1,2,3" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Step 2: Parse Selection

Accept input formats:
- Comma-separated: `1,2,3`
- Ranges: `1-3`
- Mixed: `1-3,5`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-6 only, deduplicate, sort.

### Step 3: Launch Selected Review Agents

Launch agents based on selected focuses, all in parallel:
- **Focuses 1, 2, 3, 5, 6** → Launch `code-reviewer` agent with focus injected
- **Focus 4 (Security)** → Launch `security-auditor` agent (already specialized)

**Agent Prompt Format for code-reviewer**: Include the focus assignment in each agent's prompt:
> Your assigned review focus is: **[Focus Name]**
>
> [Include the FULL focus definition from the reference below]
>
> Review the implementation for: [feature description]
>
> Files to review: [files modified in Phase 6 and 7]

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

**For Focus 4 (Security)**: Launch `security-auditor` agent directly (it's already specialized for OWASP Top 10, threat modeling, etc.). No focus injection needed.

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Present Full Agent Output

**IMPORTANT**: Present the FULL output from each review agent to the user - do NOT summarize or condense their findings. The user needs complete visibility into each reviewer's analysis, reasoning, and specific concerns to make informed decisions about which issues to address.

### Step 4.5: Verify Findings

For each focus that reported issues, launch an `issue-verifier` agent in parallel to validate the findings.

**Agent Prompt Format**:
> You are verifying issues found by the **[Focus Name]** review.
>
> **Issues to verify**:
> 1. [FILE:LINE] Description - {confidence}%
> 2. [FILE:LINE] Description - {confidence}%
> ...
>
> **Files to examine**: [list of files mentioned in the issues]
>
> For each issue, determine if it is:
> - **confirmed**: Real issue that should be fixed
> - **false-positive**: Not actually an issue (explain why)
> - **uncertain**: Needs human judgment
>
> Read the code, examine surrounding context, and provide a clear verdict with reasoning for each issue.

Launch one `issue-verifier` agent per focus that had findings, all in parallel.

**WAIT for ALL verification agents** - Do NOT proceed until every launched agent has returned.

### Step 5: Reconcile with Plan (Verified Issues)

1. Read `claude-tmp/devflow-plan.md` and extract existing `REVIEW-NNN` tasks
2. **Process verification results**:
   - **Filter out** issues marked as `false-positive` (remove from consideration)
   - **Keep** issues marked as `confirmed` (include in normal findings)
   - **Flag** issues marked as `uncertain` (include with `[NEEDS REVIEW]` marker)
3. Map remaining high-confidence issues (>=80 for code-reviewer, >=85 for security-auditor) to actionable tasks
4. Propose updates to `REVIEW-NNN` tasks:
   - Add specific issue-fixing tasks (e.g., "REVIEW-003: Fix null check in auth.ts:45")
   - Refine generic review tasks with specific findings
   - Note any REVIEW-NNN tasks that are no longer relevant
5. Deduplicate overlapping issues (same file + line + similar description)
6. Organize by severity:
   - **Critical Issues** (Confidence 90-100): Must be fixed
   - **Important Issues** (Confidence 80-89): Should be addressed
   - **Suggestions** (Confidence < 80): Nice to have improvements

### Step 6: Present Reconciled Review Tasks

Display verification statistics and verified findings:
```
## Verification Summary

- **Confirmed**: N issues (will be presented for fixing)
- **False Positives**: N issues (filtered out)
- **Needs Review**: N issues (uncertain, flagged for human judgment)

### Filtered as False Positives
- [FILE:LINE] Description - Reason: [brief explanation from verifier]
- ...

---

## Review Findings Summary

### Original REVIEW Tasks
- REVIEW-001: [description] - [keep/modify/complete]
- REVIEW-002: [description] - [keep/modify/complete]

### Proposed New REVIEW Tasks (from verified findings)
- REVIEW-003: [specific issue from findings]
- REVIEW-004: [specific issue from findings]

### Critical Issues ({count})
1. [FILE:LINE] Description - {confidence}% → **confirmed**
   Reasoning: [verifier explanation]
2. ...

### Important Issues ({count})
1. [FILE:LINE] Description - {confidence}% → **confirmed**
   Reasoning: [verifier explanation]
2. ...

### Needs Review ({count}) [UNCERTAIN]
1. [FILE:LINE] Description - {confidence}% → **uncertain**
   Reasoning: [why this needs human judgment]
2. ...

### Suggestions ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...
```

**Note**: Issues marked `[UNCERTAIN]` require your judgment. The verifier could not definitively confirm or rule out these issues - review the reasoning and decide whether to fix.

### Step 7: User Selection

Ask the user which issues to fix:

```
Which issues should I fix?

Enter numbers separated by commas (e.g., "1,3,5"), "all" to fix everything, or "skip" to proceed to summary:
```

Wait for user response and parse their input.

### Step 8: Update Plan File

1. Update `claude-tmp/devflow-plan.md`:
   - Add/modify REVIEW-NNN tasks based on user selection
   - Each selected issue becomes a trackable task with sequential ID
   - Mark any skipped original REVIEW-NNN tasks as complete (if user chose to skip)
2. Add progress log entry: `| [timestamp] | Review tasks refined based on reviewer output |`
3. Add progress log entry: `| [timestamp] | User selected {count} issues to fix |`

### Step 9: Execute Review Tasks

For each selected REVIEW-NNN task in the updated plan:
1. Update state file with `currentTask: "REVIEW-NNN"`
2. Apply the fix
3. Mark task complete: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]`
4. Add progress log entry: `| [timestamp] | REVIEW-NNN completed |`

### Step 10: Offer Re-review

If any fixes were applied, ask:

```
Fixes applied. Would you like to re-run the review to verify?

Type: "review" to run again, or "proceed" to continue to summary:
```

If user types "review", return to Step 1 with the same or different focus selection.

**Plan File Updates Summary**:
- At phase start: Add `| [timestamp] | Quality review initiated |` to progress log
- After reconciliation: Add `| [timestamp] | Review tasks refined based on reviewer output |`
- After user selection: Add `| [timestamp] | User selected {count} issues to fix |`
- After each fix: Mark `REVIEW-NNN` task complete with `Completed: [timestamp]`, add progress log entry
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
- Delete state file (`claude-tmp/devflow-feature-state.json`)
- **Only delete plan file if ALL tasks and acceptance criteria are complete**
- If any tasks are incomplete, keep `devflow-plan.md` as a record of unfinished work

**Output**: Completion summary with next steps

---

## Usage

This workflow is invoked with `/feature` followed by a feature description:

### Basic Usage
```
/feature Add user authentication with OAuth support
/feature
```

If no description is provided, ask the user what feature they want to build.

**Note**: All agent selection phases are interactive:
- Phase 2: Exploration focuses
- Phase 4: Architecture perspectives
- Phase 7: Test focuses
- Phase 8: Review focuses

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
