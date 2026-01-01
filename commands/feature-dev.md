---
description: Guided feature development with codebase understanding and architecture focus
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite, AskUser
argument-hint: [--explorers=N] [--architects=N] [--analyzers=N] [--reviewers=N] <feature-description>
---

# Feature Development Workflow

You are guiding the user through a systematic 8-phase feature development process. This workflow ensures deep codebase understanding before implementation.

## Core Principles

1. **Ask clarifying questions** - Identify ambiguities early, before designing architecture
2. **Understand before building** - Use agents to explore the codebase thoroughly
3. **Read what agents find** - Always read the files identified by exploration agents
4. **Keep code simple** - Follow existing patterns, avoid over-engineering

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

**Output**: User-selected architecture blueprint

---

## Phase 5: Implementation

**Goal**: Build the feature following the approved architecture

**Actions**:
1. **Only begin after explicit user approval of architecture**
2. Follow the implementation sequence from the blueprint
3. For each component:
   - Read existing similar code for patterns
   - Implement following established conventions
   - Keep changes minimal and focused
4. Use TodoWrite to track implementation progress
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

**Output**: Implemented feature with passing existing tests

---

## Phase 6: Testing

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
   - If all tests pass, proceed to Phase 7
   - If tests fail:
     - Review the failure report from `test-runner`
     - Fix the failing tests (adjust test code or implementation as needed)
     - Re-run `test-runner` until all pass

**CRITICAL**: Do NOT write tests until user has made an explicit selection via `AskUserQuestion`.

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

## Phase 7: Quality Review

**Goal**: Ensure code quality and correctness

**Actions**:
1. Launch {reviewers} `code-reviewer` agent(s) in parallel:
   - **If 1**: Comprehensive review covering all aspects
   - **If 2**: Split between (1) correctness/bugs and (2) conventions/maintainability
   - **If 3+**: Distribute across correctness, conventions, and maintainability
2. Collect review findings
3. Address critical and important issues
4. Re-review if significant changes were made
5. **Optional**: If user requests security audit, launch `security-auditor` agent

**Output**: Quality-verified implementation

---

## Phase 8: Summary

**Goal**: Document what was accomplished

**Actions**:
1. List all files created or modified
2. Summarize key architectural decisions made
3. Document test coverage achieved
4. Note any deferred work or known limitations
5. Suggest potential follow-up tasks
6. Mark all todos as complete

**Output**: Completion summary with next steps

---

## Usage

This workflow is invoked with `/feature-dev` followed by optional flags and a feature description:

### Basic Usage
```
/feature-dev Add user authentication with OAuth support
/feature-dev
```

### With Agent Count Overrides
```
/feature-dev --explorers=2 --architects=2 Add user authentication
/feature-dev --reviewers=5 Implement payment processing
/feature-dev --explorers=1 --architects=1 --reviewers=1 Small utility function
```

### Flag Reference

| Flag | Default | Range | Phase |
|------|---------|-------|-------|
| `--explorers=N` | 3 | 1-5 | Phase 2: Codebase Exploration |
| `--architects=N` | 3 | 1-5 | Phase 4: Architecture Design |
| `--analyzers=N` | 1 | 1-5 | Phase 6: Testing |
| `--reviewers=N` | 3 | 1-5 | Phase 7: Quality Review |

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
