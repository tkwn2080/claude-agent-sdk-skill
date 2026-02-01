# Hooks Reference

Complete reference for the Claude Agent SDK hooks system.

## Overview

Hooks are middleware functions that intercept agent execution at specific lifecycle points. They enable:
- Blocking dangerous operations
- Auditing and logging
- Modifying tool inputs
- Custom permission logic
- State management

## Hook Events

| Event | When | Can Block | Python | TypeScript |
|-------|------|-----------|--------|------------|
| `PreToolUse` | Before tool executes | Yes | Yes | Yes |
| `PostToolUse` | After tool succeeds | No | Yes | Yes |
| `PostToolUseFailure` | After tool fails | No | Partial* | Yes |
| `UserPromptSubmit` | User submits prompt | No | Yes | Yes |
| `Stop` | Agent stops | No | Yes | Yes |
| `SubagentStart` | Subagent begins | No | No | Yes |
| `SubagentStop` | Subagent completes | No | Yes | Yes |
| `PreCompact` | Before context compaction | No | Yes | Yes |
| `Notification` | Status messages | No | No | Yes |
| `SessionStart` | Session begins | No | No | Yes |
| `SessionEnd` | Session ends | No | No | Yes |
| `PermissionRequest` | Permission prompt shown | No | No | Yes |

**Python SDK** supports 6 hooks: PreToolUse, PostToolUse, UserPromptSubmit, Stop, SubagentStop, PreCompact.

**TypeScript SDK** supports all 12 hooks above. The TypeScript-only events (Notification, SessionStart, SessionEnd, SubagentStart, PermissionRequest) are useful for UI integration, session lifecycle management, and custom permission UX.

\*PostToolUseFailure: Present in Python SDK `types.py` (`HookEvent` Literal union) and defines `PostToolUseFailureHookInput`, but not yet in the official `HookInput` union type or documentation. May work in recent SDK versions (>= ~0.1.26). Test with your specific version before relying on it.

## Hook Callback Signature

**Python:**
```python
async def hook_callback(
    input_data: dict[str, Any],
    tool_use_id: str,
    context: dict[str, Any]
) -> dict[str, Any]:
    # input_data contains:
    #   - hook_event_name: str
    #   - tool_name: str (for tool hooks)
    #   - tool_input: dict (for tool hooks)
    #
    # Returns:
    #   - {} for no action
    #   - {"hookSpecificOutput": {...}} for decisions
    pass
```

**TypeScript:**
```typescript
type HookCallback = (
  input: HookInput,
  toolUseId: string,
  context: { signal: AbortSignal }
) => Promise<HookOutput>;
```

## Permission Decision Flow

Hooks returning permission decisions are checked in order:

1. **deny** - Any deny = immediate block
2. **ask** - Requires user approval
3. **allow** - Auto-approve
4. **Default to ask** - If no decision returned

## Permission Interactions with Hooks

### Full Permission Evaluation Order

When a tool is invoked, permissions are evaluated in this order:

1. **Hooks** (PreToolUse) - First checked. Can `allow`, `deny`, or `ask`.
2. **Permission Rules** - Static rules from configuration.
3. **Permission Mode** - `default`, `acceptEdits`, `plan`, or `bypassPermissions`.
4. **`can_use_tool` callback** - Final fallback for programmatic decisions.

If no step produces a decision, the default behavior is to prompt the user for permission.

### What `acceptEdits` covers (and doesn't)

`acceptEdits` auto-approves:
- File operations: `Edit`, `Write`, `NotebookEdit`
- Filesystem Bash commands: `mkdir`, `rm`, `mv`, `cp`, `touch`

`acceptEdits` does **NOT** auto-approve:
- MCP tools (`mcp__*`)
- Arbitrary Bash commands (non-filesystem)
- `WebFetch`, `WebSearch`

For MCP tools in `acceptEdits` mode, you need one of:
- A PreToolUse hook returning `allow` for those tools
- The `can_use_tool` callback
- `bypassPermissions` mode

### The `can_use_tool` callback

The `can_use_tool` callback provides programmatic permission decisions at runtime. It requires streaming input mode (async generator prompt) because it automatically sets `permission_prompt_tool_name="stdio"`, enabling the control protocol over stdin/stdout.

```python
async def permission_handler(tool_name, tool_input, context):
    if tool_name.startswith("mcp__myserver__"):
        return PermissionResultAllow(updated_input=tool_input)
    return PermissionResultDeny(message=f"Tool {tool_name} not authorized")
```

**Known issue**: Always pass `updated_input=tool_input` (the original arguments) in `PermissionResultAllow`. Some SDK versions send empty arguments if `updated_input` is omitted. (Python SDK #320)

### Headless mode gotcha

Without `can_use_tool` or explicit PreToolUse `allow` decisions, any tool that requires permission will trigger the CLI's terminal UI prompt. In headless/SDK mode (no terminal), this causes:

- **"Stream closed"** errors in hook callbacks
- Permission request loops (Claude Code #14229)
- Silent tool failures

**Recommendation for headless Python agents with MCP tools**: Use PreToolUse hooks to explicitly `allow` known MCP tools. This is more reliable than `can_use_tool` because it avoids the streaming input requirement.

```python
async def allow_mcp_tools(input_data, tool_use_id, context):
    tool_name = input_data.get("tool_name", "")
    if tool_name.startswith("mcp__myserver__"):
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "allow"
            }
        }
    return {}

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [HookMatcher(matcher="mcp__.*", hooks=[allow_mcp_tools])]
    }
)
```

## PreToolUse Hooks

Execute before a tool runs. Can block, allow, or modify input.

### Block Dangerous Commands

**Python:**
```python
from claude_agent_sdk import HookMatcher

BLOCKED_PATTERNS = [
    "rm -rf /",
    "rm -rf ~",
    "sudo rm",
    "chmod 777",
    "> /dev/sda",
    "mkfs",
    "dd if=/dev/zero"
]

async def block_dangerous_bash(input_data, tool_use_id, context):
    if input_data.get("tool_name") != "Bash":
        return {}

    command = input_data.get("tool_input", {}).get("command", "")

    for pattern in BLOCKED_PATTERNS:
        if pattern in command:
            return {
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "deny",
                    "permissionDecisionReason": f"Blocked dangerous pattern: {pattern}"
                }
            }

    return {}

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [HookMatcher(matcher="Bash", hooks=[block_dangerous_bash])]
    }
)
```

**TypeScript:**
```typescript
const blockDangerousBash: HookCallback = async (input, toolUseId, { signal }) => {
  if (input.hook_event_name !== "PreToolUse") return {};

  const preInput = input as PreToolUseHookInput;
  if (preInput.tool_name !== "Bash") return {};

  const command = (preInput.tool_input as any).command ?? "";
  const blockedPatterns = ["rm -rf /", "sudo rm", "chmod 777"];

  for (const pattern of blockedPatterns) {
    if (command.includes(pattern)) {
      return {
        hookSpecificOutput: {
          hookEventName: "PreToolUse",
          permissionDecision: "deny" as const,
          permissionDecisionReason: `Blocked: ${pattern}`
        }
      };
    }
  }

  return {};
};
```

### Protect Sensitive Files

**Python:**
```python
PROTECTED_FILES = [".env", ".env.local", "secrets.json", "credentials.json"]

async def protect_sensitive_files(input_data, tool_use_id, context):
    tool_input = input_data.get("tool_input", {})
    file_path = tool_input.get("file_path", "")

    filename = file_path.split("/")[-1]

    if filename in PROTECTED_FILES:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": f"Protected file: {filename}"
            }
        }

    return {}

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [HookMatcher(matcher="Write|Edit|Read", hooks=[protect_sensitive_files])]
    }
)
```

### Require Confirmation for Git Push

**Python:**
```python
async def confirm_git_push(input_data, tool_use_id, context):
    if input_data.get("tool_name") != "Bash":
        return {}

    command = input_data.get("tool_input", {}).get("command", "")

    if "git push" in command:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "ask",
                "permissionDecisionReason": f"Confirm git push: {command}"
            }
        }

    return {}
```

### Modify Tool Input

Redirect file operations to sandbox:

**TypeScript:**
```typescript
const sandboxWrites: HookCallback = async (input, toolUseId, { signal }) => {
  if (input.hook_event_name !== "PreToolUse") return {};

  const preInput = input as PreToolUseHookInput;
  if (!["Write", "Edit"].includes(preInput.tool_name)) return {};

  const originalPath = (preInput.tool_input as any).file_path as string;

  // Skip if already in sandbox
  if (originalPath.startsWith("/sandbox/")) return {};

  return {
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "allow" as const,
      updatedInput: {
        ...preInput.tool_input,
        file_path: `/sandbox${originalPath}`
      }
    }
  };
};
```

## PostToolUse Hooks

Execute after successful tool completion. Cannot block.

### Audit Logging

**Python:**
```python
import json
from datetime import datetime

async def audit_logger(input_data, tool_use_id, context):
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "event": input_data.get("hook_event_name"),
        "tool": input_data.get("tool_name"),
        "tool_use_id": tool_use_id,
        "input": input_data.get("tool_input"),
        "output": input_data.get("tool_output")  # Available in PostToolUse
    }

    with open("./agent_audit.jsonl", "a") as f:
        f.write(json.dumps(log_entry) + "\n")

    return {}

options = ClaudeAgentOptions(
    hooks={
        "PostToolUse": [HookMatcher(hooks=[audit_logger])]
    }
)
```

### Track File Changes

**TypeScript:**
```typescript
import { appendFileSync } from "fs";

const trackFileChanges: HookCallback = async (input, toolUseId, { signal }) => {
  if (input.hook_event_name !== "PostToolUse") return {};

  const postInput = input as PostToolUseHookInput;

  if (["Write", "Edit"].includes(postInput.tool_name)) {
    const filePath = (postInput.tool_input as any).file_path;
    const entry = `${new Date().toISOString()}: Modified ${filePath}\n`;
    appendFileSync("./file_changes.log", entry);
  }

  return {};
};
```

## PostToolUseFailure Hooks

Execute after a tool fails. Useful for error tracking and recovery logic.

**Python SDK status**: The Python SDK `types.py` on `main` branch now includes `PostToolUseFailure` in the `HookEvent` Literal union and defines `PostToolUseFailureHookInput`. However, the official documentation still marks it as TypeScript-only. If using Python SDK >= ~0.1.26, it may work. Test with your specific version.

**TypeScript:**
```typescript
const handleToolFailure: HookCallback = async (input, toolUseId, { signal }) => {
  const toolName = (input as any).tool_name;
  const error = (input as any).tool_output?.error ?? "Unknown error";

  appendFileSync(
    "./tool_failures.jsonl",
    JSON.stringify({
      timestamp: new Date().toISOString(),
      event: "tool_failure",
      tool: toolName,
      error,
      toolUseId
    }) + "\n"
  );

  return {};
};
```

**Python (experimental - test with your SDK version):**
```python
async def handle_tool_failure(input_data, tool_use_id, context):
    tool_name = input_data.get("tool_name", "unknown")
    error = input_data.get("tool_output", {}).get("error", "Unknown error")

    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "event": "tool_failure",
        "tool": tool_name,
        "error": error,
        "tool_use_id": tool_use_id
    }

    with open("./tool_failures.jsonl", "a") as f:
        f.write(json.dumps(log_entry) + "\n")

    return {}

# Note: May not work in all Python SDK versions
options = ClaudeAgentOptions(
    hooks={
        "PostToolUseFailure": [HookMatcher(hooks=[handle_tool_failure])]
    }
)
```

## PermissionRequest Hooks (TypeScript Only)

Execute when a permission prompt is shown to the user. Useful for custom permission UX.

```typescript
const onPermissionRequest: HookCallback = async (input, toolUseId, { signal }) => {
  // Log permission requests for audit
  console.log(`Permission requested for tool: ${(input as any).tool_name}`);
  return {};
};

const options = {
  hooks: {
    PermissionRequest: [{ hooks: [onPermissionRequest] }]
  }
};
```

## Stop Hooks

Execute when agent stops (success or error).

### Save Session State

**Python:**
```python
import json

async def save_session_state(input_data, tool_use_id, context):
    session_data = {
        "timestamp": datetime.now().isoformat(),
        "result": input_data.get("result"),
        "duration_ms": input_data.get("duration_ms"),
        "total_cost_usd": input_data.get("total_cost_usd")
    }

    with open("./session_history.jsonl", "a") as f:
        f.write(json.dumps(session_data) + "\n")

    return {}

options = ClaudeAgentOptions(
    hooks={
        "Stop": [HookMatcher(hooks=[save_session_state])]
    }
)
```

## SubagentStop Hooks

Execute when a subagent completes. Useful for aggregation.

**Python:**
```python
async def aggregate_subagent_results(input_data, tool_use_id, context):
    subagent_name = input_data.get("subagent_name")
    result = input_data.get("result")

    # Store in shared context or external storage
    if not hasattr(aggregate_subagent_results, "results"):
        aggregate_subagent_results.results = []

    aggregate_subagent_results.results.append({
        "agent": subagent_name,
        "result": result
    })

    return {}
```

## UserPromptSubmit Hooks

Execute when user submits a prompt. Inject context or validate.

**Python:**
```python
async def inject_context(input_data, tool_use_id, context):
    # Add system context to all prompts
    return {
        "hookSpecificOutput": {
            "hookEventName": "UserPromptSubmit",
            "additionalContext": "Remember: Always follow security best practices."
        }
    }
```

## Chaining Multiple Hooks

Hooks execute in order. Each can pass or block.

**Python:**
```python
options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            # First: Rate limiting
            HookMatcher(hooks=[rate_limiter]),

            # Second: Authorization
            HookMatcher(matcher="Bash", hooks=[check_bash_authorization]),
            HookMatcher(matcher="Write|Edit", hooks=[check_file_authorization]),

            # Third: Input sanitization
            HookMatcher(hooks=[sanitize_inputs]),

            # Last: Audit logging
            HookMatcher(hooks=[audit_pre_tool])
        ],
        "PostToolUse": [
            HookMatcher(hooks=[audit_post_tool]),
            HookMatcher(matcher="Write|Edit", hooks=[track_file_changes])
        ]
    }
)
```

## HookMatcher Options

```python
# Match all tools
HookMatcher(hooks=[my_hook])

# Match specific tools (regex)
HookMatcher(matcher="Bash", hooks=[bash_hook])
HookMatcher(matcher="Write|Edit", hooks=[file_hook])
HookMatcher(matcher="mcp__.*", hooks=[mcp_hook])

# Multiple matchers
HookMatcher(matcher="Read|Glob|Grep", hooks=[readonly_hook])
```

## Async Operations in Hooks

Hooks support async operations for external calls.

**Python:**
```python
import aiohttp

async def notify_webhook(input_data, tool_use_id, context):
    if input_data.get("hook_event_name") != "PostToolUse":
        return {}

    try:
        async with aiohttp.ClientSession() as session:
            await session.post(
                "https://api.example.com/webhook",
                json={
                    "tool": input_data.get("tool_name"),
                    "timestamp": datetime.now().isoformat()
                },
                timeout=aiohttp.ClientTimeout(total=5)
            )
    except Exception as e:
        # Log but don't fail the agent
        print(f"Webhook failed: {e}")

    return {}
```

## Error Handling in Hooks

Hooks should handle errors gracefully to avoid breaking the agent.

**TypeScript:**
```typescript
const safeAuditHook: HookCallback = async (input, toolUseId, { signal }) => {
  try {
    // Audit logic
    appendFileSync("./audit.log", JSON.stringify(input) + "\n");
  } catch (error) {
    // Log error but don't throw
    console.error("Audit hook failed:", error);
  }

  // Always return valid output
  return {};
};
```

## Common Patterns

### Rate Limiting

```python
import time
from collections import defaultdict

call_counts = defaultdict(list)
RATE_LIMIT = 10  # calls per minute

async def rate_limiter(input_data, tool_use_id, context):
    tool = input_data.get("tool_name")
    now = time.time()

    # Remove old entries
    call_counts[tool] = [t for t in call_counts[tool] if now - t < 60]

    if len(call_counts[tool]) >= RATE_LIMIT:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": f"Rate limit exceeded for {tool}"
            }
        }

    call_counts[tool].append(now)
    return {}
```

### Cost Tracking

```python
async def track_costs(input_data, tool_use_id, context):
    if input_data.get("hook_event_name") != "Stop":
        return {}

    cost = input_data.get("total_cost_usd", 0)

    # Store cost
    with open("./costs.jsonl", "a") as f:
        f.write(json.dumps({
            "timestamp": datetime.now().isoformat(),
            "cost_usd": cost
        }) + "\n")

    return {}
```

### Tool Usage Analytics

```python
from collections import Counter

tool_usage = Counter()

async def track_tool_usage(input_data, tool_use_id, context):
    if input_data.get("hook_event_name") == "PostToolUse":
        tool = input_data.get("tool_name")
        tool_usage[tool] += 1

    return {}
```

## Troubleshooting Hook Errors

### "Stream closed" during hook callbacks

**Symptom**: Hook callbacks receive "Stream closed" errors, especially for MCP tools or permission-related hooks.

**Root cause (permissions)**: The CLI uses its terminal UI for permission prompts. In headless/SDK mode (no terminal attached), the prompt transport fails because there's no stdin/stdout terminal to communicate with. This affects:
- MCP tools not covered by `acceptEdits`
- Any tool that triggers a permission prompt without an explicit `allow` decision

**Workaround**: Use PreToolUse hooks to return explicit `allow` decisions for all tools that would otherwise trigger a permission prompt. See the "Permission Interactions with Hooks" section above.

**Root cause (generator exhaustion)**: When using `query()` with an async generator for `prompt`, the SDK's `stream_input()` method closes stdin after the generator exhausts + a 60-second timeout (`CLAUDE_CODE_STREAM_CLOSE_TIMEOUT`). For long-running agents, this causes "Stream closed" errors on hook callbacks and `can_use_tool` responses even when permissions are correctly configured.

**Fix**: Keep the generator alive until the agent completes:

```python
import asyncio
from claude_agent_sdk import query, ResultMessage

stream_done = asyncio.Event()

async def generate_messages():
    yield {
        "type": "user",
        "message": {"role": "user", "content": task_prompt},
    }
    # Keeps stream_input() in its async-for loop so stdin stays open
    await stream_done.wait()

try:
    async for message in query(prompt=generate_messages(), options=options):
        if isinstance(message, ResultMessage):
            result_message = message
finally:
    stream_done.set()  # Unblock generator → stdin closes cleanly
```

**Why it works**: `stream_input()` iterates the generator with `async for`. While the generator is suspended on `await stream_done.wait()`, the loop never exits, so stdin stays open. When `query()` finishes iterating (after `ResultMessage`), the `finally` block sets the event, the generator returns, and `stream_input` proceeds to close stdin cleanly after `_first_result_event` is already set.

**References**: [Claude Code #9705](https://github.com/anthropics/claude-code/issues/9705), [Claude Code #14229](https://github.com/anthropics/claude-code/issues/14229), [anthropic-sdk-typescript #840](https://github.com/anthropics/anthropic-sdk-typescript/issues/840)

### `can_use_tool` empty arguments bug

**Symptom**: When using the `can_use_tool` callback, the `tool_input` argument arrives as an empty dict `{}` even though the tool was called with arguments.

**Root cause**: Python SDK bug where `PermissionResultAllow` without `updated_input` causes the original arguments to be dropped.

**Workaround**: Always pass through the original input in your allow response:

```python
async def permission_handler(tool_name, tool_input, context):
    # IMPORTANT: Always include updated_input with the original arguments
    return PermissionResultAllow(updated_input=tool_input)

    # NOT this - arguments will be empty:
    # return PermissionResultAllow()
```

**Reference**: Python SDK #320

### Hook errors don't crash the agent

Hook callback errors are caught by the SDK and don't terminate the agent process. However, they can cause:
- **Tool failures**: If a PreToolUse hook errors out, the tool may proceed without the expected permission decision, potentially triggering a terminal permission prompt (which fails in headless mode).
- **Infinite retry loops**: If a hook consistently fails, Claude may retry the tool call repeatedly, burning through turns and budget.
- **Silent data loss**: PostToolUse audit hooks that error won't log the tool execution.

**Best practice**: Always wrap hook logic in try/except and return `{}` on error to ensure graceful degradation.
