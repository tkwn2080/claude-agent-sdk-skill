---
name: claude-agent-sdk
description: >
  Build production AI agents with the Claude Agent SDK. Use when working with:
  (1) Creating autonomous agents with query() or ClaudeSDKClient,
  (2) Defining custom tools via in-process MCP servers,
  (3) Implementing hooks for tool interception and permissions,
  (4) Building multi-agent systems with subagents,
  (5) Managing sessions and state across queries,
  (6) Integrating MCP servers for external capabilities.
  Covers Python and TypeScript SDKs with production patterns.
---

# Claude Agent SDK

The Agent SDK provides Claude with autonomous tool execution. Unlike the Messages API where you implement the tool loop, agents execute tools automatically.

## Quick Start

**Python:**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find and fix the bug in auth.py",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Bash"])
    ):
        print(message)

asyncio.run(main())
```

**TypeScript:**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix the bug in auth.py",
  options: { allowedTools: ["Read", "Edit", "Bash"] }
})) {
  console.log(message);
}
```

## When to Use Agent SDK vs Messages API

| Scenario | Use |
|----------|-----|
| Autonomous multi-step tasks | Agent SDK |
| File operations, code editing | Agent SDK |
| Simple Q&A, text generation | Messages API |
| Custom tool execution control | Messages API |
| Browser automation, web tasks | Agent SDK |

## Core APIs

### `query()` - One-off Tasks
Creates new session each time. Best for independent tasks.

```python
async for message in query(
    prompt="Your task here",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Write", "Edit", "Bash"],
        permission_mode="acceptEdits"  # Auto-approve edits
    )
):
    if hasattr(message, 'result'):
        print(message.result)
```

### `ClaudeSDKClient` - Conversations
Maintains context across exchanges. Use for interactive applications.

```python
async with ClaudeSDKClient(options=options) as client:
    await client.query("Read the auth module")
    async for msg in client.receive_response():
        print(msg)

    await client.query("Now refactor it")  # Remembers context
    async for msg in client.receive_response():
        print(msg)
```

## Built-in Tools

| Tool | Purpose |
|------|---------|
| `Read` | Read files in working directory |
| `Write` | Create new files |
| `Edit` | Make precise edits to existing files |
| `Bash` | Run terminal commands and scripts |
| `Glob` | Find files by pattern (`**/*.ts`) |
| `Grep` | Search file contents with regex |
| `WebSearch` | Search the web |
| `WebFetch` | Fetch and parse web pages |
| `AskUserQuestion` | Request user clarification |
| `Task` | Spawn subagents |

## Custom Tools

Custom tools use in-process MCP servers. Tool names follow: `mcp__{server}__{tool}`

**Python:**
```python
from claude_agent_sdk import tool, create_sdk_mcp_server

@tool("get_weather", "Get temperature for coordinates",
      {"latitude": float, "longitude": float})
async def get_weather(args):
    # Fetch weather data
    return {"content": [{"type": "text", "text": f"72°F"}]}

server = create_sdk_mcp_server(
    name="weather",
    version="1.0.0",
    tools=[get_weather]
)

options = ClaudeAgentOptions(
    mcp_servers={"weather": server},
    allowed_tools=["mcp__weather__get_weather"]
)
```

**TypeScript:**
```typescript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const server = createSdkMcpServer({
  name: "weather",
  version: "1.0.0",
  tools: [
    tool("get_weather", "Get temperature", {
      latitude: z.number(),
      longitude: z.number()
    }, async (args) => ({
      content: [{ type: "text", text: "72°F" }]
    }))
  ]
});
```

**Important**: Custom MCP tools require streaming input mode (async generator).

See [custom-tools.md](references/custom-tools.md) for complete patterns.

## Hooks System

Hooks intercept agent execution at lifecycle points.

| Hook | When | Use Case |
|------|------|----------|
| `PreToolUse` | Before tool executes | Block dangerous commands |
| `PostToolUse` | After tool succeeds | Audit logging |
| `Stop` | Agent stops | Save session state |
| `SubagentStop` | Subagent completes | Aggregate results |

### Permission Decisions

Hooks can return permission decisions (checked in order):
1. **deny** - Block immediately
2. **ask** - Require user approval
3. **allow** - Auto-approve

**Example: Block .env modifications**
```python
async def protect_env(input_data, tool_use_id, context):
    file_path = input_data['tool_input'].get('file_path', '')
    if file_path.endswith('.env'):
        return {
            'hookSpecificOutput': {
                'hookEventName': 'PreToolUse',
                'permissionDecision': 'deny',
                'permissionDecisionReason': 'Cannot modify .env files'
            }
        }
    return {}

options = ClaudeAgentOptions(
    hooks={'PreToolUse': [HookMatcher(matcher='Write|Edit', hooks=[protect_env])]}
)
```

See [hooks-reference.md](references/hooks-reference.md) for all hooks and patterns.

## Subagents

Spawn specialized agents for focused subtasks with parallelization.

```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Task"],
    agents={
        "code-reviewer": AgentDefinition(
            description="Expert code reviewer for quality and security",
            prompt="Analyze code quality and suggest improvements",
            tools=["Read", "Glob", "Grep"]
        ),
        "test-writer": AgentDefinition(
            description="Test generation specialist",
            prompt="Write comprehensive unit tests",
            tools=["Read", "Write", "Bash"]
        )
    }
)
```

See [multi-agent.md](references/multi-agent.md) for orchestration patterns.

## Session Management

Resume sessions to maintain context across queries:

```python
session_id = None

# First query: capture session ID
async for message in query(prompt="Read auth.py", options=options):
    if hasattr(message, 'subtype') and message.subtype == 'init':
        session_id = message.session_id

# Resume with full context
async for message in query(
    prompt="Find all callers",  # "auth.py" understood from context
    options=ClaudeAgentOptions(resume=session_id)
):
    print(message)
```

## Configuration Options

```python
ClaudeAgentOptions(
    allowed_tools=["Read", "Edit"],      # Whitelist tools
    permission_mode="acceptEdits",        # default|acceptEdits|plan|bypassPermissions
    mcp_servers={"name": server},         # External MCP servers
    cwd="/path/to/project",               # Working directory
    max_turns=10,                         # Prevent infinite loops
    resume="session-id",                  # Resume session
    hooks={...},                          # Lifecycle hooks
    agents={...},                         # Subagent definitions
    model="claude-sonnet-4-20250514",     # Model selection
    env={"API_KEY": "..."}                # Environment variables
)
```

## Security Best Practices

1. **Deny-by-default**: Only whitelist necessary tools
2. **Confirm sensitive actions**: Use hooks for git push, infrastructure changes
3. **Block dangerous commands**: rm -rf, sudo, etc.
4. **Protect secrets**: Never expose credentials in agent-visible context
5. **Use short-lived tokens**: Rotate credentials aggressively
6. **Log all tool invocations**: Audit trail with timestamps

## Error Handling

```python
from claude_agent_sdk import (
    CLINotFoundError,      # Claude Code not installed
    CLIConnectionError,    # Connection issues
    ProcessError,          # Process failed
    CLIJSONDecodeError     # JSON parsing issues
)

try:
    async for message in query(prompt="Task", options=options):
        pass
except ProcessError as e:
    print(f"Failed with exit code: {e.exit_code}")
```

## Message Types

```python
# Stream yields different message types
async for message in query(prompt="Task", options=options):
    if message.type == "assistant":
        print(message.content)
    elif message.type == "result":
        if message.subtype == "success":
            print(f"Completed: {message.result}")
        print(f"Cost: ${message.total_cost_usd}")
```

## Authentication

- **Anthropic API**: `export ANTHROPIC_API_KEY=your-key`
- **Amazon Bedrock**: `CLAUDE_CODE_USE_BEDROCK=1` + AWS credentials
- **Google Vertex AI**: `CLAUDE_CODE_USE_VERTEX=1` + GCP credentials

## Reference Files

- [Python Patterns](references/python-patterns.md) - Complete Python examples
- [TypeScript Patterns](references/typescript-patterns.md) - Complete TypeScript examples
- [Hooks Reference](references/hooks-reference.md) - All hooks with examples
- [Custom Tools](references/custom-tools.md) - Tool creation guide
- [Multi-Agent Patterns](references/multi-agent.md) - Orchestration patterns
