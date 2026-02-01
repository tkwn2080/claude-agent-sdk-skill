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
    await client.connect()                    # Establish connection
    await client.query("Read the auth module")
    async for msg in client.receive_messages():  # Stream all messages
        print(msg)

    await client.query("Now refactor it")     # Remembers context
    async for msg in client.receive_messages():
        print(msg)

    await client.interrupt()                  # Cancel current operation
    await client.rewind_files()               # Restore file checkpoints
    status = await client.get_mcp_status()    # Check MCP server status
    await client.disconnect()                 # Clean up
```

## Built-in Tools

| Tool | Purpose |
|------|---------|
| `Read` | Read files in working directory |
| `Write` | Create new files |
| `Edit` | Make precise edits to existing files |
| `Bash` | Run terminal commands and scripts |
| `BashOutput` | Retrieve output from background bash shells |
| `KillBash` | Kill running background bash shells |
| `Glob` | Find files by pattern (`**/*.ts`) |
| `Grep` | Search file contents with regex |
| `NotebookEdit` | Edit Jupyter notebook cells |
| `WebSearch` | Search the web |
| `WebFetch` | Fetch and parse web pages |
| `AskUserQuestion` | Request user clarification |
| `TodoWrite` | Create/manage structured task lists |
| `Task` | Spawn subagents |
| `ExitPlanMode` | Exit planning mode |
| `ListMcpResources` | List available MCP resources |
| `ReadMcpResource` | Read content from MCP resources |

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

| Hook | When | Python | TypeScript |
|------|------|--------|------------|
| `PreToolUse` | Before tool executes | Yes | Yes |
| `PostToolUse` | After tool succeeds | Yes | Yes |
| `PostToolUseFailure` | After tool fails | Partial* | Yes |
| `UserPromptSubmit` | User submits prompt | Yes | Yes |
| `Stop` | Agent stops | Yes | Yes |
| `SubagentStop` | Subagent completes | Yes | Yes |
| `PreCompact` | Before context compaction | Yes | Yes |
| `Notification` | Status messages | No | Yes |
| `SessionStart` | Session begins | No | Yes |
| `SessionEnd` | Session ends | No | Yes |
| `SubagentStart` | Subagent begins | No | Yes |
| `PermissionRequest` | Permission prompt shown | No | Yes |

\*PostToolUseFailure: Present in Python SDK `types.py` (`HookEvent` Literal) but not yet in the official `HookInput` union type or documentation. May work in recent SDK versions (>= ~0.1.26).

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
            tools=["Read", "Glob", "Grep"],
            model="sonnet"  # "sonnet" | "opus" | "haiku" | "inherit"
        ),
        "test-writer": AgentDefinition(
            description="Test generation specialist",
            prompt="Write comprehensive unit tests",
            tools=["Read", "Write", "Bash"],
            model="sonnet"
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
    model="claude-sonnet-4-5-20250929",   # Model selection (or "claude-sonnet-4-5", "claude-opus-4-5", "claude-haiku-4-5")
    fallback_model="claude-haiku-4-5",    # Fallback if primary model unavailable
    system_prompt="Custom instructions",  # Override system prompt
    max_budget_usd=1.0,                   # Maximum spend limit
    max_thinking_tokens=10000,            # Extended thinking budget
    enable_file_checkpointing=True,       # Enable file state snapshots for rewind
    output_format={"type": "json", "schema": {...}},  # Structured JSON output
    plugins=[...],                        # Plugin extensions
    sandbox=True,                         # Run in sandboxed environment
    setting_sources=[...],               # Custom setting sources
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
- **Microsoft Azure AI Foundry**: `CLAUDE_CODE_USE_FOUNDRY=1` + Azure credentials

### OpenRouter

Route Agent SDK requests through [OpenRouter](https://openrouter.ai) for access to multiple model providers, fallbacks, and cost optimization:

```bash
export ANTHROPIC_BASE_URL="https://openrouter.ai/api"
export ANTHROPIC_AUTH_TOKEN="$OPENROUTER_API_KEY"
export ANTHROPIC_API_KEY=""  # Must be explicitly empty
```

The Agent SDK inherits Claude Code's model override environment variables, so you can route to different OpenRouter models:

```bash
# Route to specific models via OpenRouter
export ANTHROPIC_DEFAULT_SONNET_MODEL="anthropic/claude-sonnet-4"
export ANTHROPIC_DEFAULT_OPUS_MODEL="anthropic/claude-opus-4"
```

**Important**: `ANTHROPIC_API_KEY` must be set to an empty string — if omitted entirely, the SDK will fail with an authentication error. The actual auth is handled by `ANTHROPIC_AUTH_TOKEN`.

## Structured Outputs

Use `output_format` for validated JSON responses:

```python
options = ClaudeAgentOptions(
    output_format={
        "type": "json",
        "schema": {
            "type": "object",
            "properties": {
                "summary": {"type": "string"},
                "issues": {"type": "array", "items": {"type": "string"}},
                "severity": {"type": "string", "enum": ["low", "medium", "high"]}
            },
            "required": ["summary", "issues", "severity"]
        }
    }
)
```

## File Checkpointing

Enable file state snapshots for safe rollback:

```python
options = ClaudeAgentOptions(
    enable_file_checkpointing=True,
    allowed_tools=["Read", "Edit", "Write"]
)

# Later, rewind all file changes made by the agent
async with ClaudeSDKClient(options=options) as client:
    await client.connect()
    await client.query("Refactor the auth module")
    async for msg in client.receive_messages():
        pass
    # If results are unsatisfactory:
    await client.rewind_files()  # Restores all files to pre-query state
```

## Extended Thinking

Enable deeper reasoning with thinking token budgets:

```python
options = ClaudeAgentOptions(
    max_thinking_tokens=10000,  # Budget for extended thinking
    allowed_tools=["Read", "Glob", "Grep"]
)
```

## TypeScript Query Object Methods

The TypeScript SDK's `Query` object (returned by `query()`) exposes additional control methods:

```typescript
const q = query({ prompt: "Analyze codebase", options });

// Control methods
await q.interrupt();              // Cancel current operation
await q.rewindFiles();            // Restore file checkpoints
await q.setPermissionMode("acceptEdits");
await q.setModel("claude-opus-4-5");
await q.setMaxThinkingTokens(10000);

// Informational methods
const commands = await q.supportedCommands();
const models = await q.supportedModels();
const mcpStatus = await q.mcpServerStatus();
const account = await q.accountInfo();
```

## Known Issues & Troubleshooting

### In-process MCP servers unreliable with subagents

`create_sdk_mcp_server()` can fail with "Stream closed" when used with parallel subagents due to message-queue backpressure. The SDK's internal transport gets blocked when multiple subagents compete for the MCP connection.

**Recommendation**: Use stdio subprocess MCP servers for production multi-agent setups. In-process servers are fine for single-agent simple use cases. (Python SDK #425, TS SDK #41)

### `acceptEdits` does NOT auto-approve MCP tools

The `acceptEdits` permission mode only auto-approves file operations (`Edit`, `Write`) and filesystem Bash commands (`mkdir`, `rm`, `mv`, `cp`). MCP tools still require explicit permission resolution.

**Options for MCP tool approval:**
- `bypassPermissions` - Approves everything (use with caution)
- `can_use_tool` callback - Programmatic per-tool decisions (requires streaming input mode)
- PreToolUse hooks returning `allow` - Most reliable for headless agents

### Hook "Stream closed" in headless mode

Without a `can_use_tool` callback, the CLI uses its terminal UI for permission prompts. In headless/SDK mode this fails with "Stream closed" because there is no terminal to display the prompt.

**Workaround**: Use PreToolUse hooks to return explicit `allow`/`deny` decisions for all tools that would otherwise trigger a permission prompt. This avoids the terminal UI path entirely. See [hooks-reference.md](references/hooks-reference.md) for permission interaction details.

## Reference Files

- [Python Patterns](references/python-patterns.md) - Complete Python examples
- [TypeScript Patterns](references/typescript-patterns.md) - Complete TypeScript examples
- [Hooks Reference](references/hooks-reference.md) - All hooks with examples
- [Custom Tools](references/custom-tools.md) - Tool creation guide
- [Multi-Agent Patterns](references/multi-agent.md) - Orchestration patterns
