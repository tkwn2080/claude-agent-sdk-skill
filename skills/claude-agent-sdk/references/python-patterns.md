# Python Patterns

Complete Python patterns for the Claude Agent SDK.

## Installation

```bash
pip install claude-agent-sdk
```

Requirements: Python 3.10+

## Setup

```python
import os
os.environ["ANTHROPIC_API_KEY"] = "your-api-key"
```

Or via shell:
```bash
export ANTHROPIC_API_KEY=your-api-key
```

## Basic Query Pattern

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Analyze the codebase structure",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep"],
            cwd="/path/to/project"
        )
    ):
        # Handle different message types
        if hasattr(message, 'content'):
            for block in message.content:
                if hasattr(block, 'text'):
                    print(block.text)

        if hasattr(message, 'result'):
            print(f"\nResult: {message.result}")
            print(f"Duration: {message.duration_ms}ms")
            if message.total_cost_usd:
                print(f"Cost: ${message.total_cost_usd:.4f}")

asyncio.run(main())
```

## ClaudeSDKClient for Conversations

```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

async def interactive_session():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
        permission_mode="acceptEdits"
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.connect()

        # First exchange
        await client.query("Read and summarize the main module")
        async for msg in client.receive_messages():
            print_message(msg)

        # Follow-up with context
        await client.query("What are the key functions?")
        async for msg in client.receive_messages():
            print_message(msg)

        # Another follow-up
        await client.query("Add docstrings to the public functions")
        async for msg in client.receive_messages():
            print_message(msg)

        await client.disconnect()

def print_message(msg):
    if hasattr(msg, 'content'):
        for block in msg.content:
            if hasattr(block, 'text'):
                print(block.text, end='', flush=True)
    if hasattr(msg, 'result'):
        print(f"\n[Completed: {msg.result}]")

asyncio.run(interactive_session())
```

## Session Resumption

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def resumable_workflow():
    session_id = None

    # Phase 1: Analysis
    async for message in query(
        prompt="Analyze the authentication system",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Glob", "Grep"])
    ):
        if hasattr(message, 'subtype') and message.subtype == 'init':
            session_id = message.session_id
            print(f"Session started: {session_id}")
        if hasattr(message, 'result'):
            print(f"Analysis complete: {message.result}")

    # Phase 2: Resume and extend
    async for message in query(
        prompt="Now identify security vulnerabilities",
        options=ClaudeAgentOptions(resume=session_id)
    ):
        if hasattr(message, 'result'):
            print(f"Security review: {message.result}")

    # Phase 3: Resume and fix
    async for message in query(
        prompt="Fix the identified vulnerabilities",
        options=ClaudeAgentOptions(
            resume=session_id,
            allowed_tools=["Read", "Edit", "Bash"]
        )
    ):
        if hasattr(message, 'result'):
            print(f"Fixes applied: {message.result}")

asyncio.run(resumable_workflow())
```

## Streaming Input with Custom Tools

Custom MCP tools require streaming input:

```python
import asyncio
from typing import AsyncIterator, Any
from claude_agent_sdk import query, ClaudeAgentOptions, tool, create_sdk_mcp_server

@tool("lookup_user", "Look up user by ID", {"user_id": str})
async def lookup_user(args: dict[str, Any]) -> dict[str, Any]:
    # Simulate database lookup
    users = {"123": "Alice", "456": "Bob"}
    name = users.get(args["user_id"], "Unknown")
    return {
        "content": [{"type": "text", "text": f"User: {name}"}]
    }

async def generate_messages() -> AsyncIterator[dict[str, Any]]:
    yield {
        "type": "user",
        "message": {
            "role": "user",
            "content": "Look up user 123"
        }
    }

async def main():
    server = create_sdk_mcp_server(
        name="users",
        version="1.0.0",
        tools=[lookup_user]
    )

    async for message in query(
        prompt=generate_messages(),  # Streaming input
        options=ClaudeAgentOptions(
            mcp_servers={"users": server},
            allowed_tools=["mcp__users__lookup_user"],
            max_turns=5
        )
    ):
        if hasattr(message, 'result'):
            print(message.result)

asyncio.run(main())
```

## Subagent Definitions

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def multi_agent_review():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep", "Task"],
        agents={
            "security-auditor": AgentDefinition(
                description="Security specialist for vulnerability detection",
                prompt="Analyze code for security issues: injection, auth bypass, data exposure",
                tools=["Read", "Glob", "Grep"]
            ),
            "performance-analyst": AgentDefinition(
                description="Performance optimization specialist",
                prompt="Identify performance bottlenecks and optimization opportunities",
                tools=["Read", "Glob", "Grep"]
            ),
            "documentation-checker": AgentDefinition(
                description="Documentation quality reviewer",
                prompt="Check docstrings, comments, and README completeness",
                tools=["Read", "Glob", "Grep"]
            )
        }
    )

    async for message in query(
        prompt="Review the entire codebase using all available specialist agents",
        options=options
    ):
        if hasattr(message, 'result'):
            print(f"Review complete: {message.result}")

asyncio.run(multi_agent_review())
```

## Hooks with Type Hints

```python
import asyncio
from typing import Any
from datetime import datetime
from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher

async def audit_hook(
    input_data: dict[str, Any],
    tool_use_id: str,
    context: dict[str, Any]
) -> dict[str, Any]:
    """Log all tool invocations to audit file."""
    import json

    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "tool": input_data.get("tool_name"),
        "event": input_data.get("hook_event_name"),
        "tool_use_id": tool_use_id
    }

    with open("./agent_audit.log", "a") as f:
        f.write(json.dumps(log_entry) + "\n")

    return {}

async def rate_limiter(
    input_data: dict[str, Any],
    tool_use_id: str,
    context: dict[str, Any]
) -> dict[str, Any]:
    """Rate limit Bash commands."""
    if input_data.get("tool_name") != "Bash":
        return {}

    # Check rate limit (simplified)
    import time
    current_time = time.time()

    # In production, use Redis or similar
    if not hasattr(rate_limiter, "last_call"):
        rate_limiter.last_call = 0

    if current_time - rate_limiter.last_call < 1.0:  # 1 second limit
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Rate limited: wait 1 second"
            }
        }

    rate_limiter.last_call = current_time
    return {}

async def main():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Bash"],
        hooks={
            "PreToolUse": [
                HookMatcher(hooks=[rate_limiter]),
                HookMatcher(hooks=[audit_hook])
            ],
            "PostToolUse": [
                HookMatcher(hooks=[audit_hook])
            ]
        }
    )

    async for message in query(prompt="List files and read README", options=options):
        if hasattr(message, 'result'):
            print(message.result)

asyncio.run(main())
```

## Error Handling Pattern

```python
import asyncio
from claude_agent_sdk import (
    query,
    ClaudeAgentOptions,
    CLINotFoundError,
    CLIConnectionError,
    ProcessError,
    CLIJSONDecodeError
)

async def robust_agent():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Bash"],
        max_turns=10  # Prevent infinite loops
    )

    try:
        async for message in query(prompt="Refactor the utils module", options=options):
            if hasattr(message, 'result'):
                if message.subtype == "success":
                    print(f"Success: {message.result}")
                elif message.subtype == "error":
                    print(f"Agent error: {message.result}")

    except CLINotFoundError:
        print("Error: Claude Code CLI not installed")
        print("Install with: pip install claude-agent-sdk")

    except CLIConnectionError as e:
        print(f"Connection error: {e}")
        print("Check your API key and network connection")

    except ProcessError as e:
        print(f"Process failed with exit code: {e.exit_code}")
        if e.stderr:
            print(f"Error output: {e.stderr}")

    except CLIJSONDecodeError as e:
        print(f"Failed to parse response: {e}")

    except Exception as e:
        print(f"Unexpected error: {type(e).__name__}: {e}")

asyncio.run(robust_agent())
```

## Permission Modes

```python
from claude_agent_sdk import ClaudeAgentOptions

# Default: Ask for permission on sensitive operations
default_mode = ClaudeAgentOptions(
    permission_mode="default",
    allowed_tools=["Read", "Edit", "Bash"]
)

# Accept edits: Auto-approve file modifications
auto_edit_mode = ClaudeAgentOptions(
    permission_mode="acceptEdits",
    allowed_tools=["Read", "Edit", "Write"]
)

# Plan mode: Agent creates plan, doesn't execute
plan_mode = ClaudeAgentOptions(
    permission_mode="plan",
    allowed_tools=["Read", "Glob", "Grep"]
)

# Bypass: No permission prompts (use with caution)
bypass_mode = ClaudeAgentOptions(
    permission_mode="bypassPermissions",
    allowed_tools=["Read", "Edit", "Bash"]
)
```

## Permission Callbacks (`can_use_tool`)

The `can_use_tool` callback provides programmatic permission decisions for tools not covered by `permission_mode`. It requires streaming input mode (async generator prompt).

```python
import asyncio
from claude_agent_sdk import (
    ClaudeAgentOptions, query,
    PermissionResultAllow, PermissionResultDeny
)

async def permission_handler(tool_name, tool_input, context):
    """Handle permission requests for tools not covered by permission_mode."""
    # Allow known MCP tools
    if tool_name.startswith("mcp__myserver__"):
        # IMPORTANT: Always pass through original input (Python SDK #320 workaround)
        return PermissionResultAllow(updated_input=tool_input)

    # Allow specific built-in tools
    if tool_name in ("WebSearch", "WebFetch"):
        return PermissionResultAllow(updated_input=tool_input)

    # Deny unknown tools
    return PermissionResultDeny(message=f"Tool {tool_name} not authorized")

# Requires streaming input mode
async def generate_prompt():
    yield {
        "type": "user",
        "message": {
            "role": "user",
            "content": "Use the MCP tool to query the database"
        }
    }
    # Keep open until result received (see custom-tools.md Known Issues)

options = ClaudeAgentOptions(
    allowed_tools=["Read", "mcp__myserver__query"],
    permission_mode="acceptEdits",
    can_use_tool=permission_handler,  # Auto-sets permission_prompt_tool_name="stdio"
)

async def main():
    async for message in query(prompt=generate_prompt(), options=options):
        if hasattr(message, 'result'):
            print(message.result)

asyncio.run(main())
```

**Alternative: PreToolUse hooks** - Use PreToolUse hooks to `allow` MCP tools instead of `can_use_tool`. The hook path avoids the streaming input requirement and is generally more reliable for headless agents. See [hooks-reference.md](hooks-reference.md) for the pattern.

## Environment Variables

```python
from claude_agent_sdk import ClaudeAgentOptions

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Bash"],
    env={
        "NODE_ENV": "development",
        "DATABASE_URL": "postgres://localhost/dev",
        "API_KEY": os.environ.get("MY_API_KEY", "")
    }
)
```

## Working Directory Control

```python
from pathlib import Path
from claude_agent_sdk import ClaudeAgentOptions

# String path
options1 = ClaudeAgentOptions(
    cwd="/home/user/projects/myapp",
    allowed_tools=["Read", "Edit"]
)

# Path object
options2 = ClaudeAgentOptions(
    cwd=Path.home() / "projects" / "myapp",
    allowed_tools=["Read", "Edit"]
)
```

## Model Selection

```python
from claude_agent_sdk import ClaudeAgentOptions

# Use specific model (pinned version)
options = ClaudeAgentOptions(
    model="claude-sonnet-4-5-20250929",  # or claude-opus-4-5-20251101
    allowed_tools=["Read", "Edit", "Bash"]
)

# Use short alias
options = ClaudeAgentOptions(
    model="claude-sonnet-4-5",  # or "claude-opus-4-5", "claude-haiku-4-5"
    allowed_tools=["Read", "Edit", "Bash"]
)

# With fallback model
options = ClaudeAgentOptions(
    model="claude-opus-4-5",
    fallback_model="claude-sonnet-4-5",
    allowed_tools=["Read", "Edit", "Bash"]
)
```

## Extended Options

```python
from claude_agent_sdk import ClaudeAgentOptions

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Bash"],
    system_prompt="You are a senior code reviewer",
    max_budget_usd=2.0,
    max_thinking_tokens=10000,
    enable_file_checkpointing=True,
    output_format={
        "type": "json",
        "schema": {
            "type": "object",
            "properties": {
                "summary": {"type": "string"},
                "issues": {"type": "array"}
            }
        }
    }
)
```

## ClaudeSDKClient Methods

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

async def full_client_example():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Bash"],
        enable_file_checkpointing=True
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.connect()                     # Establish connection

        await client.query("Refactor auth module")
        async for msg in client.receive_messages(): # Stream messages
            print_message(msg)

        await client.interrupt()                   # Cancel current operation
        await client.rewind_files()                # Restore file checkpoints
        status = await client.get_mcp_status()     # Check MCP server status
        await client.disconnect()                  # Clean up
```
