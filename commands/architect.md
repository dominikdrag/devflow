---
description: Architecture design with codebase exploration, clarification, and parallel architect agents
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite
argument-hint: <feature-description>
---

# Architecture Design Workflow

You are guiding the user through a systematic 7-phase architecture design process. This workflow produces a comprehensive architecture design without implementation.

## Core Principles

1. **Ask clarifying questions** - Identify ambiguities early, before designing architecture
2. **Understand before designing** - Use agents to explore the codebase thoroughly
3. **Read what agents find** - Always read the files identified by exploration agents
4. **Present full proposals** - Never summarize architect agent output

---

## Workflow State Management

This workflow uses a state file (`claude-tmp/architect-state.json`) to persist progress across conversation compaction and session interruptions.

### On Workflow Start

**FIRST**, check if `claude-tmp/architect-state.json` exists:
- **If file exists and `active: true`**: This is a RESUMED workflow
  - Read the state file to understand current progress
  - Inform the user: "Resuming architect workflow from Phase {currentPhase}"
  - Display historical context from `phaseHistory`:
    - For each completed phase, show phase name and key outputs
    - Phase 2: Show key files and patterns discovered
    - Phase 3: Show clarifications made
    - Phase 5/6: Show selected perspectives and architecture
  - Continue from the current phase (do NOT restart from Phase 1)
- **If file does not exist**: This is a NEW workflow
  - Create initial state file:
    ```json
    {
      "active": true,
      "sessionId": "architect-{timestamp}",
      "featureDescription": "[from user input]",
      "currentPhase": 1,
      "phaseHistory": [],
      "explorationFocuses": [],
      "perspectivesSelected": [],
      "selectedArchitecture": null,
      "startedAt": "[current ISO timestamp]",
      "lastUpdatedAt": "[current ISO timestamp]"
    }
    ```
  - Proceed with Phase 1

### State File Format

```json
{
  "active": true,
  "sessionId": "architect-{timestamp}",
  "featureDescription": "...",
  "currentPhase": 1,
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
  "explorationFocuses": [1, 2, 3],
  "perspectivesSelected": [1, 2, 3],
  "selectedArchitecture": null,
  "startedAt": "ISO timestamp",
  "lastUpdatedAt": "ISO timestamp"
}
```

### Phase-Specific Outputs

Each phase stores structured outputs in `phaseHistory[].outputs`:

| Phase | Name | Outputs |
|-------|------|---------|
| 1 | Discovery | `requirements[]`, `constraints[]` |
| 2 | Codebase Exploration | `focusesSelected[]`, `keyFiles[]`, `patterns[]`, `integrationPoints[]` |
| 3 | Clarifying Questions | `clarifications[]` (array of `{question, answer}`) |
| 4 | Perspective Selection | `perspectivesSelected[]` |
| 5 | Launch Architects | `proposalsReceived[]` |
| 6 | Present Proposals | `selectedArchitecture`, `selectionRationale` |
| 7 | Summary | `architectureDocumented`, `nextStepsSuggested` |

### Updating State

At the START of each phase, update the state file:
- Set `currentPhase` to the new phase number
- Set `lastUpdatedAt` to current ISO timestamp
- Add new entry to `phaseHistory[]` with `status: "in_progress"`, `startedAt`, and empty `outputs`

At the END of each phase, update the state file:
- Update the phase's `phaseHistory` entry: set `status: "completed"`, `completedAt`, and populate `outputs`

---

## Configuration

### Parse Arguments

Arguments: $ARGUMENTS

The entire argument text is the feature description. If no description is provided, ask the user what they want to architect.

---

## Phase 1: Discovery

**Goal**: Understand what needs to be architected

**Actions**:
1. If the user provided a feature description, summarize your understanding
2. Identify the core requirements and constraints
3. Ask initial clarifying questions if the request is ambiguous
4. Confirm understanding before proceeding

**Output**: Clear statement of what will be designed

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

## Phase 4: Perspective Selection

**Goal**: Select architecture perspectives for parallel architect agents

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

**Output**: List of selected perspective numbers for Phase 5

---

## Phase 5: Launch Architect Agents

**Goal**: Generate architecture proposals from multiple perspectives

**Actions**:

### Step 1: Launch Selected Architects

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

**Output**: All architect agent proposals collected

---

## Phase 6: Present Proposals & User Selection

**Goal**: Present all proposals and get user's architecture selection

**Actions**:

### Step 1: Present All Proposals

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

### Step 2: User Selection

Ask the user to select an architecture by typing their choice:

```
Which architecture do you want to use?

Enter a number (1-N), or type "custom" to describe your own approach:
```

Wait for user response. Parse their input:
- If a number: select that architecture
- If "custom": wait for user to describe their approach, then summarize and confirm

Document the selected architecture before proceeding.

**CRITICAL**: Do NOT proceed to Phase 7 until user has typed their selection. The response IS the approval gate.

**Output**: User-selected architecture blueprint

---

## Phase 7: Summary

**Goal**: Document the selected architecture and suggest next steps

**Actions**:

### Step 1: Document Selected Architecture

Create a comprehensive summary of the selected architecture:

```
## Architecture Design Summary

### Feature
[Feature description from Phase 1]

### Selected Approach: [Perspective Name]

**Philosophy**: [How the perspective shapes this design]

**Architecture Diagram**:
```
[ASCII diagram showing NEW and EXISTING components]
```

**Key Architectural Decisions**:
1. [Decision 1 with rationale]
2. [Decision 2 with rationale]
3. [Decision 3 with rationale]

**Component Design**:
| Component | Responsibility | Dependencies | Interface |
|-----------|----------------|--------------|-----------|
| [Name] | [What it does] | [What it needs] | [Public API] |

**Data Flow**: [How information moves through the system]

**Files to Create/Modify**:
- `path/to/file.ts` - [purpose and what it will contain]
- `path/to/existing/file.ts` - [what specific changes will be made]

**Trade-offs Acknowledged**:
- **Pros**: [advantages of this approach]
- **Cons**: [disadvantages/risks accepted]

**Critical Considerations**:
- **Error Handling**: [approach]
- **State Management**: [approach]
- **Testing Strategy**: [suggested approach]
- **Performance**: [considerations]
- **Security**: [considerations]
```

### Step 2: Suggest Next Steps

Provide actionable next steps:

```
## Next Steps

This architecture design is complete. To implement:

1. **Run `/feature [feature-description]`** - Use the full feature workflow with this architecture in mind
   - During Phase 4 (Architecture Design), you can reference or reuse this design
   - The workflow will guide you through planning, implementation, testing, and review

2. **Run `/tdd [feature-description]`** - Use test-driven development with this architecture
   - Tests will be designed first based on the component interfaces defined above
   - Implementation follows the TDD Red-Green-Refactor cycle

3. **Manual implementation** - If you prefer to implement without a guided workflow:
   - Start with the files listed in "Files to Create/Modify"
   - Follow the build sequence from the architecture proposal
   - Use the component design table as a reference
```

### Step 3: Cleanup

1. Delete state file (`claude-tmp/architect-state.json`)
2. Mark all todos as complete

**Output**: Complete architecture documentation with next steps

---

## Usage

This workflow is invoked with `/architect` followed by a feature description:

### Basic Usage
```
/architect Add user authentication with OAuth support
/architect
```

If no description is provided, ask the user what feature they want to architect.

## When to Use This Workflow

**Use for**:
- Architecture exploration before committing to implementation
- Comparing multiple design approaches for complex features
- Getting expert perspectives on architectural decisions
- Understanding trade-offs between different approaches
- Pre-planning for features that will be implemented later

**Don't use for**:
- Simple features with obvious architecture
- Trivial changes with clear implementation paths
- When you're ready to implement immediately (use `/feature` or `/tdd` instead)
