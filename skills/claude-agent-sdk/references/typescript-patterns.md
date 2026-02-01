# TypeScript Patterns

Complete TypeScript patterns for the Claude Agent SDK.

## Installation

```bash
npm install @anthropic-ai/claude-agent-sdk
# or
bun add @anthropic-ai/claude-agent-sdk
```

## Setup

```typescript
// Set via environment
process.env.ANTHROPIC_API_KEY = "your-api-key";

// Or export in shell
// export ANTHROPIC_API_KEY=your-api-key
```

## Basic Query Pattern

```typescript
import { query, type ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

const options: ClaudeAgentOptions = {
  allowedTools: ["Read", "Glob", "Grep"],
  cwd: "/path/to/project"
};

for await (const message of query({ prompt: "Analyze the codebase", options })) {
  if ("content" in message) {
    for (const block of message.content) {
      if ("text" in block) {
        process.stdout.write(block.text);
      }
    }
  }

  if ("result" in message) {
    console.log(`\nResult: ${message.result}`);
    console.log(`Duration: ${message.duration_ms}ms`);
    if (message.total_cost_usd) {
      console.log(`Cost: $${message.total_cost_usd.toFixed(4)}`);
    }
  }
}
```

## ClaudeSDKClient for Conversations

```typescript
import { ClaudeSDKClient, type ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

async function interactiveSession() {
  const options: ClaudeAgentOptions = {
    allowedTools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
    permissionMode: "acceptEdits"
  };

  const client = new ClaudeSDKClient({ options });

  try {
    await client.connect();

    // First exchange
    await client.query("Read and summarize the main module");
    for await (const msg of client.receiveResponse()) {
      printMessage(msg);
    }

    // Follow-up with context
    await client.query("What are the key functions?");
    for await (const msg of client.receiveResponse()) {
      printMessage(msg);
    }

    // Another follow-up
    await client.query("Add docstrings to the public functions");
    for await (const msg of client.receiveResponse()) {
      printMessage(msg);
    }

  } finally {
    await client.disconnect();
  }
}

function printMessage(msg: any) {
  if ("content" in msg) {
    for (const block of msg.content) {
      if ("text" in block) {
        process.stdout.write(block.text);
      }
    }
  }
  if ("result" in msg) {
    console.log(`\n[Completed: ${msg.result}]`);
  }
}

interactiveSession();
```

## Session Resumption

```typescript
import { query, type ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

async function resumableWorkflow() {
  let sessionId: string | undefined;

  // Phase 1: Analysis
  for await (const message of query({
    prompt: "Analyze the authentication system",
    options: { allowedTools: ["Read", "Glob", "Grep"] }
  })) {
    if ("subtype" in message && message.subtype === "init") {
      sessionId = message.session_id;
      console.log(`Session started: ${sessionId}`);
    }
    if ("result" in message) {
      console.log(`Analysis complete: ${message.result}`);
    }
  }

  // Phase 2: Resume and extend
  for await (const message of query({
    prompt: "Now identify security vulnerabilities",
    options: { resume: sessionId }
  })) {
    if ("result" in message) {
      console.log(`Security review: ${message.result}`);
    }
  }

  // Phase 3: Resume and fix
  for await (const message of query({
    prompt: "Fix the identified vulnerabilities",
    options: {
      resume: sessionId,
      allowedTools: ["Read", "Edit", "Bash"]
    }
  })) {
    if ("result" in message) {
      console.log(`Fixes applied: ${message.result}`);
    }
  }
}

resumableWorkflow();
```

## Custom Tools with Zod Schemas

```typescript
import { query, tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

// Define tool with Zod schema
const weatherTool = tool(
  "get_weather",
  "Get current temperature for coordinates",
  {
    latitude: z.number().describe("Latitude coordinate"),
    longitude: z.number().describe("Longitude coordinate"),
    units: z.enum(["celsius", "fahrenheit"]).optional().default("fahrenheit")
  },
  async (args) => {
    const response = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${args.latitude}&longitude=${args.longitude}&current=temperature_2m`
    );
    const data = await response.json();

    return {
      content: [{
        type: "text" as const,
        text: `Temperature: ${data.current.temperature_2m}°${args.units === "celsius" ? "C" : "F"}`
      }]
    };
  }
);

const server = createSdkMcpServer({
  name: "weather",
  version: "1.0.0",
  tools: [weatherTool]
});

// Streaming input required for custom tools
async function* generateMessages() {
  yield {
    type: "user" as const,
    message: {
      role: "user" as const,
      content: "What's the weather in San Francisco (37.7749, -122.4194)?"
    }
  };
}

for await (const message of query({
  prompt: generateMessages(),
  options: {
    mcpServers: { weather: server },
    allowedTools: ["mcp__weather__get_weather"],
    maxTurns: 5
  }
})) {
  if ("result" in message) {
    console.log(message.result);
  }
}
```

## Complex Tool: API Gateway

```typescript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const apiGatewayTool = tool(
  "api_request",
  "Make authenticated API requests to external services",
  {
    service: z.enum(["stripe", "github", "slack"]),
    endpoint: z.string(),
    method: z.enum(["GET", "POST", "PUT", "DELETE"]),
    body: z.record(z.any()).optional(),
    query: z.record(z.string()).optional()
  },
  async (args) => {
    const configs: Record<string, { baseUrl: string; key: string }> = {
      stripe: {
        baseUrl: "https://api.stripe.com/v1",
        key: process.env.STRIPE_KEY ?? ""
      },
      github: {
        baseUrl: "https://api.github.com",
        key: process.env.GITHUB_TOKEN ?? ""
      },
      slack: {
        baseUrl: "https://slack.com/api",
        key: process.env.SLACK_TOKEN ?? ""
      }
    };

    const { baseUrl, key } = configs[args.service];
    const url = new URL(`${baseUrl}${args.endpoint}`);

    if (args.query) {
      Object.entries(args.query).forEach(([k, v]) => url.searchParams.set(k, v));
    }

    try {
      const response = await fetch(url, {
        method: args.method,
        headers: {
          Authorization: `Bearer ${key}`,
          "Content-Type": "application/json"
        },
        body: args.body ? JSON.stringify(args.body) : undefined
      });

      const data = await response.json();
      return {
        content: [{
          type: "text" as const,
          text: JSON.stringify(data, null, 2)
        }]
      };
    } catch (error) {
      return {
        content: [{
          type: "text" as const,
          text: `API Error: ${(error as Error).message}`
        }]
      };
    }
  }
);

const apiServer = createSdkMcpServer({
  name: "api-gateway",
  version: "1.0.0",
  tools: [apiGatewayTool]
});
```

## Subagent Definitions

```typescript
import { query, type AgentDefinition } from "@anthropic-ai/claude-agent-sdk";

const agents: Record<string, AgentDefinition> = {
  "security-auditor": {
    description: "Security specialist for vulnerability detection",
    prompt: "Analyze code for security issues: injection, auth bypass, data exposure",
    tools: ["Read", "Glob", "Grep"],
    model: "sonnet"  // "sonnet" | "opus" | "haiku" | "inherit"
  },
  "performance-analyst": {
    description: "Performance optimization specialist",
    prompt: "Identify performance bottlenecks and optimization opportunities",
    tools: ["Read", "Glob", "Grep"],
    model: "sonnet"
  },
  "documentation-checker": {
    description: "Documentation quality reviewer",
    prompt: "Check docstrings, comments, and README completeness",
    tools: ["Read", "Glob", "Grep"],
    model: "haiku"
  }
};

for await (const message of query({
  prompt: "Review the entire codebase using all available specialist agents",
  options: {
    allowedTools: ["Read", "Glob", "Grep", "Task"],
    agents
  }
})) {
  if ("result" in message) {
    console.log(`Review complete: ${message.result}`);
  }
}
```

## Hooks with TypeScript Types

```typescript
import {
  query,
  type HookCallback,
  type PreToolUseHookInput,
  type PostToolUseHookInput
} from "@anthropic-ai/claude-agent-sdk";
import { appendFileSync } from "fs";

const auditHook: HookCallback = async (input, toolUseId, { signal }) => {
  const logEntry = {
    timestamp: new Date().toISOString(),
    tool: (input as any).tool_name,
    event: input.hook_event_name,
    toolUseId
  };

  appendFileSync("./agent_audit.log", JSON.stringify(logEntry) + "\n");
  return {};
};

const protectEnvFiles: HookCallback = async (input, toolUseId, { signal }) => {
  if (input.hook_event_name !== "PreToolUse") return {};

  const preInput = input as PreToolUseHookInput;
  const filePath = (preInput.tool_input as any).file_path ?? "";

  if (filePath.endsWith(".env")) {
    return {
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny" as const,
        permissionDecisionReason: "Cannot modify .env files"
      }
    };
  }

  return {};
};

const blockDangerousCommands: HookCallback = async (input, toolUseId, { signal }) => {
  if (input.hook_event_name !== "PreToolUse") return {};

  const preInput = input as PreToolUseHookInput;
  if (preInput.tool_name !== "Bash") return {};

  const command = (preInput.tool_input as any).command ?? "";
  const dangerous = ["rm -rf /", "sudo", "chmod 777", "> /dev/"];

  if (dangerous.some(d => command.includes(d))) {
    return {
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny" as const,
        permissionDecisionReason: `Blocked dangerous command: ${command}`
      }
    };
  }

  return {};
};

for await (const message of query({
  prompt: "Refactor the utils module",
  options: {
    allowedTools: ["Read", "Edit", "Bash"],
    hooks: {
      PreToolUse: [
        { matcher: "Bash", hooks: [blockDangerousCommands] },
        { matcher: "Write|Edit", hooks: [protectEnvFiles] },
        { hooks: [auditHook] }
      ],
      PostToolUse: [
        { hooks: [auditHook] }
      ]
    }
  }
})) {
  if ("result" in message) {
    console.log(message.result);
  }
}
```

## Modify Tool Input via Hooks

```typescript
import { type HookCallback, type PreToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

const redirectToSandbox: HookCallback = async (input, toolUseId, { signal }) => {
  if (input.hook_event_name !== "PreToolUse") return {};

  const preInput = input as PreToolUseHookInput;
  if (preInput.tool_name !== "Write") return {};

  const originalPath = (preInput.tool_input as any).file_path as string;

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

## Error Handling Pattern

```typescript
import {
  query,
  CLINotFoundError,
  CLIConnectionError,
  ProcessError,
  CLIJSONDecodeError
} from "@anthropic-ai/claude-agent-sdk";

async function robustAgent() {
  try {
    for await (const message of query({
      prompt: "Refactor the utils module",
      options: {
        allowedTools: ["Read", "Edit", "Bash"],
        maxTurns: 10
      }
    })) {
      if ("result" in message) {
        if (message.subtype === "success") {
          console.log(`Success: ${message.result}`);
        } else if (message.subtype === "error") {
          console.log(`Agent error: ${message.result}`);
        }
      }
    }

  } catch (error) {
    if (error instanceof CLINotFoundError) {
      console.error("Error: Claude Code CLI not installed");
      console.error("Install with: npm install @anthropic-ai/claude-agent-sdk");

    } else if (error instanceof CLIConnectionError) {
      console.error(`Connection error: ${error.message}`);
      console.error("Check your API key and network connection");

    } else if (error instanceof ProcessError) {
      console.error(`Process failed with exit code: ${error.exitCode}`);
      if (error.stderr) {
        console.error(`Error output: ${error.stderr}`);
      }

    } else if (error instanceof CLIJSONDecodeError) {
      console.error(`Failed to parse response: ${error.message}`);

    } else {
      console.error(`Unexpected error: ${error}`);
    }
  }
}

robustAgent();
```

## Permission Modes

```typescript
import { type ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

// Default: Ask for permission on sensitive operations
const defaultMode: ClaudeAgentOptions = {
  permissionMode: "default",
  allowedTools: ["Read", "Edit", "Bash"]
};

// Accept edits: Auto-approve file modifications
const autoEditMode: ClaudeAgentOptions = {
  permissionMode: "acceptEdits",
  allowedTools: ["Read", "Edit", "Write"]
};

// Plan mode: Agent creates plan, doesn't execute
const planMode: ClaudeAgentOptions = {
  permissionMode: "plan",
  allowedTools: ["Read", "Glob", "Grep"]
};

// Bypass: No permission prompts (use with caution)
const bypassMode: ClaudeAgentOptions = {
  permissionMode: "bypassPermissions",
  allowedTools: ["Read", "Edit", "Bash"]
};
```

## Permission Callbacks (`canUseTool`)

The `canUseTool` callback provides programmatic permission decisions for tools not covered by `permissionMode`. It requires streaming input mode (async generator prompt).

```typescript
import { query, type ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

const options: ClaudeAgentOptions = {
  allowedTools: ["Read", "mcp__myserver__query"],
  permissionMode: "acceptEdits",
  canUseTool: async (toolName: string, toolInput: Record<string, unknown>) => {
    // Allow known MCP tools
    if (toolName.startsWith("mcp__myserver__")) {
      // IMPORTANT: Always pass updatedInput with original args (SDK #320 workaround)
      return { type: "allow", updatedInput: toolInput };
    }

    // Allow specific built-in tools
    if (["WebSearch", "WebFetch"].includes(toolName)) {
      return { type: "allow", updatedInput: toolInput };
    }

    // Deny unknown tools
    return { type: "deny", message: `Tool ${toolName} not authorized` };
  }
};

// Requires streaming input mode
async function* generatePrompt() {
  yield {
    type: "user" as const,
    message: {
      role: "user" as const,
      content: "Use the MCP tool to query the database"
    }
  };
  // Keep open until result received (see custom-tools.md Known Issues)
}

for await (const message of query({ prompt: generatePrompt(), options })) {
  if ("result" in message) {
    console.log(message.result);
  }
}
```

**Alternative: PreToolUse hooks** — Use PreToolUse hooks to `allow` MCP tools instead of `canUseTool`. The hook path avoids the streaming input requirement and is generally more reliable for headless agents. See [hooks-reference.md](hooks-reference.md) for the pattern.

## External MCP Servers

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Browser automation with Playwright
for await (const message of query({
  prompt: "Open example.com and describe what you see",
  options: {
    mcpServers: {
      playwright: {
        command: "npx",
        args: ["@playwright/mcp@latest"]
      }
    }
  }
})) {
  if ("result" in message) {
    console.log(message.result);
  }
}

// Database access
for await (const message of query({
  prompt: "List all users from the database",
  options: {
    mcpServers: {
      postgres: {
        command: "npx",
        args: ["@modelcontextprotocol/server-postgres"],
        env: {
          DATABASE_URL: process.env.DATABASE_URL ?? ""
        }
      }
    }
  }
})) {
  if ("result" in message) {
    console.log(message.result);
  }
}
```

## Query Object Methods

The `Query` object returned by `query()` exposes additional control and informational methods:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const q = query({
  prompt: "Analyze codebase",
  options: {
    allowedTools: ["Read", "Glob", "Grep"],
    enableFileCheckpointing: true
  }
});

// Control methods
await q.interrupt();                          // Cancel current operation
await q.rewindFiles();                        // Restore file checkpoints
await q.setPermissionMode("acceptEdits");     // Change permission mode mid-session
await q.setModel("claude-opus-4-5");          // Switch model mid-session
await q.setMaxThinkingTokens(10000);          // Adjust thinking budget

// Informational methods
const commands = await q.supportedCommands(); // List available commands
const models = await q.supportedModels();     // List available models
const mcpStatus = await q.mcpServerStatus();  // Check MCP server health
const account = await q.accountInfo();        // Get account info
```

## Extended Options

```typescript
import { type ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

const options: ClaudeAgentOptions = {
  allowedTools: ["Read", "Edit", "Bash"],
  model: "claude-sonnet-4-5-20250929",       // Pinned model version
  fallbackModel: "claude-haiku-4-5",          // Fallback if primary unavailable
  systemPrompt: "You are a senior code reviewer",
  maxBudgetUsd: 2.0,                          // Spending limit
  maxThinkingTokens: 10000,                   // Extended thinking budget
  enableFileCheckpointing: true,              // File state snapshots
  outputFormat: {                             // Structured JSON output
    type: "json",
    schema: {
      type: "object",
      properties: {
        summary: { type: "string" },
        issues: { type: "array" }
      }
    }
  },
  plugins: [],                                // Plugin extensions
  sandbox: true,                              // Sandboxed execution
  settingSources: []                          // Custom setting sources
};
```

## Type Definitions Reference

```typescript
import type {
  ClaudeAgentOptions,
  AgentDefinition,
  HookCallback,
  HookMatcher,
  PreToolUseHookInput,
  PostToolUseHookInput,
  Message,
  AssistantMessage,
  ResultMessage,
  UserMessage,
  SystemMessage
} from "@anthropic-ai/claude-agent-sdk";
```
