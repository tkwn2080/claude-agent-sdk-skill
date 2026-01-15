# Multi-Agent Patterns

Comprehensive guide to multi-agent orchestration with the Claude Agent SDK.

## Overview

Multi-agent systems outperform single agents by:
- Parallelizing independent tasks
- Using specialized agents for focused work
- Reducing context pollution
- Enabling better error isolation

Anthropic's internal research shows **90%+ improvement** with orchestrator-worker patterns compared to single-agent approaches.

## Core Patterns

### 1. Orchestrator-Worker Pattern

Lead agent coordinates specialized subagents.

**Architecture:**
```
Lead Agent (Orchestrator)
  ├── Subagent 1 (Security Audit)
  ├── Subagent 2 (Performance Review)
  └── Subagent 3 (Documentation Check)
```

**Python:**
```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Task"],
    model="claude-opus-4-20250514",  # Lead uses stronger model
    agents={
        "security-auditor": AgentDefinition(
            description="Security specialist for vulnerability detection",
            prompt="Analyze code for security issues: SQL injection, XSS, auth bypass, data exposure. Report severity and remediation.",
            tools=["Read", "Glob", "Grep"],
            model="claude-sonnet-4-20250514"  # Subagents use faster model
        ),
        "performance-analyst": AgentDefinition(
            description="Performance optimization specialist",
            prompt="Identify performance bottlenecks: N+1 queries, memory leaks, inefficient algorithms. Suggest optimizations.",
            tools=["Read", "Glob", "Grep"],
            model="claude-sonnet-4-20250514"
        ),
        "test-coverage-checker": AgentDefinition(
            description="Test coverage and quality reviewer",
            prompt="Analyze test coverage, identify untested code paths, suggest missing test cases.",
            tools=["Read", "Glob", "Grep"],
            model="claude-sonnet-4-20250514"
        )
    }
)

async for message in query(
    prompt="""Perform a comprehensive code review of the src/ directory.

    Use the security-auditor to check for vulnerabilities.
    Use the performance-analyst to identify optimization opportunities.
    Use the test-coverage-checker to assess test quality.

    Synthesize findings into a prioritized action plan.""",
    options=options
):
    if hasattr(message, 'result'):
        print(message.result)
```

### 2. Routing Pattern

Lead agent routes to appropriate specialist based on query type.

**Python:**
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Task"],
    agents={
        "frontend-expert": AgentDefinition(
            description="Frontend specialist for React, CSS, UI/UX issues",
            prompt="Analyze and fix frontend code: React components, styling, accessibility, performance.",
            tools=["Read", "Edit", "Glob", "Grep"]
        ),
        "backend-expert": AgentDefinition(
            description="Backend specialist for API, database, server issues",
            prompt="Analyze and fix backend code: API design, database queries, authentication, caching.",
            tools=["Read", "Edit", "Glob", "Grep", "Bash"]
        ),
        "devops-expert": AgentDefinition(
            description="DevOps specialist for deployment, infrastructure, CI/CD",
            prompt="Handle DevOps tasks: Docker, Kubernetes, CI/CD pipelines, monitoring.",
            tools=["Read", "Edit", "Bash"]
        )
    }
)

async for message in query(
    prompt="The login button isn't working on mobile devices",
    options=options
):
    # Lead routes to frontend-expert based on the UI issue
    pass
```

### 3. Parallelization Pattern

Multiple subagents work on independent subtasks simultaneously.

**Python:**
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Task"],
    agents={
        "analyzer-a": AgentDefinition(
            description="Analyze module A",
            prompt="Deep analysis of module A architecture and dependencies",
            tools=["Read", "Glob", "Grep"]
        ),
        "analyzer-b": AgentDefinition(
            description="Analyze module B",
            prompt="Deep analysis of module B architecture and dependencies",
            tools=["Read", "Glob", "Grep"]
        ),
        "analyzer-c": AgentDefinition(
            description="Analyze module C",
            prompt="Deep analysis of module C architecture and dependencies",
            tools=["Read", "Glob", "Grep"]
        )
    }
)

async for message in query(
    prompt="""Analyze all three modules in parallel:
    - Use analyzer-a for src/module-a/
    - Use analyzer-b for src/module-b/
    - Use analyzer-c for src/module-c/

    After all complete, identify cross-module dependencies and integration points.""",
    options=options
):
    pass
```

### 4. Evaluator-Optimizer Pattern

One agent generates, another evaluates and refines.

**Python:**
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Write", "Edit", "Bash", "Task"],
    agents={
        "code-generator": AgentDefinition(
            description="Generate implementation code",
            prompt="Write clean, well-documented code following project conventions",
            tools=["Read", "Write", "Edit", "Glob"]
        ),
        "code-reviewer": AgentDefinition(
            description="Review and improve generated code",
            prompt="Review code for bugs, security issues, performance, style. Suggest improvements.",
            tools=["Read", "Glob", "Grep"]
        ),
        "test-writer": AgentDefinition(
            description="Write tests for new code",
            prompt="Write comprehensive unit and integration tests",
            tools=["Read", "Write", "Edit", "Bash"]
        )
    }
)

async for message in query(
    prompt="""Implement a user authentication module:

    1. Use code-generator to create the initial implementation
    2. Use code-reviewer to review and suggest improvements
    3. Apply improvements using code-generator
    4. Use test-writer to add comprehensive tests
    5. Run tests and fix any failures""",
    options=options
):
    pass
```

### 5. Prompt Chaining Pattern

Sequential agents where output feeds to next.

**Python:**
```python
# Phase 1: Research
async for msg in query(
    prompt="Research the current authentication implementation",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep"]
    )
):
    if hasattr(msg, 'session_id'):
        session_id = msg.session_id

# Phase 2: Design (using context from phase 1)
async for msg in query(
    prompt="Based on the research, design improvements for the auth system",
    options=ClaudeAgentOptions(
        resume=session_id,
        allowed_tools=["Read", "Glob", "Grep"]
    )
):
    pass

# Phase 3: Implement
async for msg in query(
    prompt="Implement the designed improvements",
    options=ClaudeAgentOptions(
        resume=session_id,
        allowed_tools=["Read", "Write", "Edit", "Bash"]
    )
):
    pass
```

## Subagent Configuration

### AgentDefinition Parameters

```python
AgentDefinition(
    description: str,     # What the agent does (shown to lead)
    prompt: str,          # System prompt for the subagent
    tools: list[str],     # Tools available to subagent
    model: str = None     # Optional model override
)
```

### Tool Access Control

Subagents should have minimal required tools:

```python
# Read-only analyzer
AgentDefinition(
    description="Code analyzer",
    prompt="Analyze code patterns",
    tools=["Read", "Glob", "Grep"]  # No write access
)

# Writer with limited scope
AgentDefinition(
    description="Test writer",
    prompt="Write unit tests",
    tools=["Read", "Write", "Bash"]  # Can run tests
)

# Full access for complex tasks
AgentDefinition(
    description="Refactoring agent",
    prompt="Refactor code with tests",
    tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
)
```

## Extended Thinking Integration

Enable deeper reasoning for complex analysis:

**TypeScript:**
```typescript
const options: ClaudeAgentOptions = {
  allowedTools: ["Read", "Glob", "Grep", "Task"],
  maxThinkingTokens: 10000,
  agents: {
    "deep-analyzer": {
      description: "Deep architectural analysis",
      prompt: "Think step by step about architecture decisions",
      tools: ["Read", "Glob", "Grep"],
      maxThinkingTokens: 5000
    }
  }
};
```

## Performance Considerations

### Token Efficiency

| Configuration | Token Usage |
|--------------|-------------|
| Single agent | Baseline |
| 2 subagents | ~3x baseline |
| 5 subagents | ~8x baseline |
| 10 subagents | ~15x baseline |

**Guidelines:**
- Use subagents for parallelizable tasks
- Prefer fewer specialized agents over many generic ones
- Use lightweight references between agents (not full outputs)

### Model Selection

```python
# Orchestrator: Use strongest model for planning
lead_model = "claude-opus-4-20250514"

# Workers: Use faster models for execution
worker_model = "claude-sonnet-4-20250514"

options = ClaudeAgentOptions(
    model=lead_model,
    agents={
        "worker-1": AgentDefinition(..., model=worker_model),
        "worker-2": AgentDefinition(..., model=worker_model)
    }
)
```

## Aggregating Results

### Using Hooks

```python
results = {}

async def capture_subagent_result(input_data, tool_use_id, context):
    if input_data.get("hook_event_name") == "SubagentStop":
        agent_name = input_data.get("subagent_name")
        result = input_data.get("result")
        results[agent_name] = result
    return {}

options = ClaudeAgentOptions(
    hooks={
        "SubagentStop": [HookMatcher(hooks=[capture_subagent_result])]
    },
    agents={...}
)
```

### Via Lead Agent Prompt

```python
async for message in query(
    prompt="""Run all three analyzers and then:

    1. Collect findings from each analyzer
    2. Deduplicate overlapping issues
    3. Prioritize by severity (critical > high > medium > low)
    4. Create a single consolidated report with:
       - Executive summary
       - Critical issues (immediate action)
       - Recommended improvements (prioritized)
       - Technical debt items (track for later)""",
    options=options
):
    pass
```

## Error Handling in Multi-Agent Systems

### Subagent Failure Isolation

```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Task"],
    agents={
        "risky-operation": AgentDefinition(
            description="Performs risky operation that might fail",
            prompt="Attempt the operation, report success or failure",
            tools=["Read", "Bash"]
        )
    }
)

async for message in query(
    prompt="""Try the risky-operation agent.
    If it fails, document the error and continue with manual fallback.
    Do not let subagent failures stop the overall workflow.""",
    options=options
):
    pass
```

### Retry Logic

```python
async def run_with_retry(prompt, options, max_retries=3):
    for attempt in range(max_retries):
        try:
            async for message in query(prompt=prompt, options=options):
                if hasattr(message, 'result') and message.subtype == 'success':
                    return message.result
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(2 ** attempt)  # Exponential backoff
```

## Real-World Example: Code Review Pipeline

```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def comprehensive_code_review(directory: str):
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep", "Task"],
        model="claude-opus-4-20250514",
        agents={
            "security": AgentDefinition(
                description="Security vulnerability scanner",
                prompt="""Scan for security issues:
                - SQL injection, XSS, CSRF
                - Hardcoded credentials
                - Insecure dependencies
                - Authentication/authorization flaws
                Report with severity, location, and fix.""",
                tools=["Read", "Glob", "Grep"],
                model="claude-sonnet-4-20250514"
            ),
            "performance": AgentDefinition(
                description="Performance issue detector",
                prompt="""Identify performance problems:
                - N+1 queries
                - Memory leaks
                - Inefficient algorithms
                - Missing indexes
                Report with impact and optimization.""",
                tools=["Read", "Glob", "Grep"],
                model="claude-sonnet-4-20250514"
            ),
            "maintainability": AgentDefinition(
                description="Code quality reviewer",
                prompt="""Review code quality:
                - Code duplication
                - Complex functions (high cyclomatic complexity)
                - Missing documentation
                - Inconsistent naming
                Report with refactoring suggestions.""",
                tools=["Read", "Glob", "Grep"],
                model="claude-sonnet-4-20250514"
            ),
            "testing": AgentDefinition(
                description="Test coverage analyzer",
                prompt="""Analyze test coverage:
                - Untested functions
                - Missing edge cases
                - Flaky tests
                - Integration test gaps
                Report with test recommendations.""",
                tools=["Read", "Glob", "Grep", "Bash"],
                model="claude-sonnet-4-20250514"
            )
        }
    )

    review_prompt = f"""Perform comprehensive code review of {directory}/

    Execute all specialist agents in parallel:
    1. security - for vulnerability scanning
    2. performance - for performance analysis
    3. maintainability - for code quality
    4. testing - for test coverage

    After all complete, synthesize into a prioritized report:

    ## Executive Summary
    (2-3 sentences on overall health)

    ## Critical Issues (fix immediately)
    (security vulnerabilities, data loss risks)

    ## High Priority (fix this sprint)
    (performance issues, major quality problems)

    ## Medium Priority (plan for next sprint)
    (maintainability, moderate issues)

    ## Low Priority (backlog)
    (minor improvements, nice-to-haves)

    ## Metrics
    - Security score: X/10
    - Performance score: X/10
    - Maintainability score: X/10
    - Test coverage: X%
    """

    async for message in query(prompt=review_prompt, options=options):
        if hasattr(message, 'content'):
            for block in message.content:
                if hasattr(block, 'text'):
                    print(block.text, end='', flush=True)
        if hasattr(message, 'result'):
            return message.result

# Usage
asyncio.run(comprehensive_code_review("src"))
```

## Best Practices

1. **Least privilege**: Give subagents minimal required tools
2. **Clear boundaries**: Define specific, non-overlapping responsibilities
3. **Fail gracefully**: Isolate subagent failures from main workflow
4. **Monitor costs**: Multi-agent uses significantly more tokens
5. **Use strong lead**: Orchestrator benefits from most capable model
6. **Parallelize wisely**: Only parallelize truly independent tasks
7. **Aggregate efficiently**: Use references, not full output copying
8. **Test individually**: Verify each subagent works before combining
