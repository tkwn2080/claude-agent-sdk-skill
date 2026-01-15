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
| `PostToolUseFailure` | After tool fails | No | No | Yes |
| `UserPromptSubmit` | User submits prompt | No | Yes | Yes |
| `Stop` | Agent stops | No | Yes | Yes |
| `SubagentStart` | Subagent begins | No | No | Yes |
| `SubagentStop` | Subagent completes | No | Yes | Yes |
| `PreCompact` | Before context compaction | No | Yes | Yes |
| `SessionStart` | Session begins | No | No | Yes |
| `SessionEnd` | Session ends | No | No | Yes |
| `Notification` | Status messages | No | No | Yes |

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
