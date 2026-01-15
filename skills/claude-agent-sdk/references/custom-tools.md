# Custom Tools Guide

Complete guide to creating custom tools for the Claude Agent SDK.

## Overview

Custom tools extend agent capabilities through in-process MCP (Model Context Protocol) servers. When Claude needs functionality beyond built-in tools, custom tools provide it.

## Tool Naming Convention

Tools are exposed to Claude with this naming pattern:

```
mcp__{server_name}__{tool_name}
```

Example:
- Server name: `weather`
- Tool name: `get_forecast`
- Full name: `mcp__weather__get_forecast`

## Basic Tool Creation

### Python

```python
from claude_agent_sdk import tool, create_sdk_mcp_server
from typing import Any

@tool("greet", "Greet a user by name", {"name": str})
async def greet(args: dict[str, Any]) -> dict[str, Any]:
    return {
        "content": [{
            "type": "text",
            "text": f"Hello, {args['name']}!"
        }]
    }

server = create_sdk_mcp_server(
    name="greeting",
    version="1.0.0",
    tools=[greet]
)
```

### TypeScript

```typescript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const greetTool = tool(
  "greet",
  "Greet a user by name",
  { name: z.string().describe("The name to greet") },
  async (args) => ({
    content: [{ type: "text" as const, text: `Hello, ${args.name}!` }]
  })
);

const server = createSdkMcpServer({
  name: "greeting",
  version: "1.0.0",
  tools: [greetTool]
});
```

## Tool Decorator/Function Parameters

### Python @tool Decorator

```python
@tool(
    name: str,           # Tool name (used in mcp__{server}__{name})
    description: str,    # Description shown to Claude
    schema: dict         # Parameter schema: {"param": type}
)
async def my_tool(args: dict[str, Any]) -> dict[str, Any]:
    pass
```

Supported schema types:
- `str` - String
- `int` - Integer
- `float` - Float
- `bool` - Boolean
- `list` - List/Array
- `dict` - Object/Dictionary

### TypeScript tool() Function

```typescript
tool(
  name: string,           // Tool name
  description: string,    // Description shown to Claude
  schema: ZodObject,      // Zod schema for parameters
  handler: (args) => Promise<ToolResult>
)
```

## Complex Parameter Schemas

### Python

```python
@tool(
    "search_users",
    "Search for users by criteria",
    {
        "query": str,
        "limit": int,
        "filters": dict,  # Nested object
        "tags": list      # Array
    }
)
async def search_users(args: dict[str, Any]) -> dict[str, Any]:
    query = args["query"]
    limit = args.get("limit", 10)
    filters = args.get("filters", {})
    tags = args.get("tags", [])

    # Search logic
    results = await perform_search(query, limit, filters, tags)

    return {
        "content": [{
            "type": "text",
            "text": json.dumps(results, indent=2)
        }]
    }
```

### TypeScript with Zod

```typescript
const searchUsersTool = tool(
  "search_users",
  "Search for users by criteria",
  {
    query: z.string().describe("Search query"),
    limit: z.number().int().min(1).max(100).optional().default(10),
    filters: z.object({
      status: z.enum(["active", "inactive", "pending"]).optional(),
      role: z.string().optional(),
      createdAfter: z.string().datetime().optional()
    }).optional(),
    tags: z.array(z.string()).optional()
  },
  async (args) => {
    const results = await performSearch(args);
    return {
      content: [{ type: "text" as const, text: JSON.stringify(results, null, 2) }]
    };
  }
);
```

## Tool Return Format

Tools must return a specific format:

```python
{
    "content": [
        {
            "type": "text",
            "text": "Result text here"
        }
    ]
}
```

Multiple content blocks:

```python
return {
    "content": [
        {"type": "text", "text": "Summary: Found 5 users"},
        {"type": "text", "text": json.dumps(users, indent=2)}
    ]
}
```

## Streaming Input Requirement

**Important**: Custom MCP tools require streaming input mode (async generator).

### Python

```python
from typing import AsyncIterator

async def generate_messages() -> AsyncIterator[dict[str, Any]]:
    yield {
        "type": "user",
        "message": {
            "role": "user",
            "content": "Use the custom tool to look up user 123"
        }
    }

async def main():
    async for message in query(
        prompt=generate_messages(),  # Streaming input
        options=ClaudeAgentOptions(
            mcp_servers={"my_server": server},
            allowed_tools=["mcp__my_server__my_tool"]
        )
    ):
        pass
```

### TypeScript

```typescript
async function* generateMessages() {
  yield {
    type: "user" as const,
    message: {
      role: "user" as const,
      content: "Use the custom tool to look up user 123"
    }
  };
}

for await (const message of query({
  prompt: generateMessages(),
  options: {
    mcpServers: { my_server: server },
    allowedTools: ["mcp__my_server__my_tool"]
  }
})) {
  // Handle messages
}
```

## Error Handling in Tools

Return errors gracefully in the content:

```python
@tool("fetch_data", "Fetch data from API", {"endpoint": str})
async def fetch_data(args: dict[str, Any]) -> dict[str, Any]:
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(args["endpoint"], timeout=10) as response:
                if response.status != 200:
                    return {
                        "content": [{
                            "type": "text",
                            "text": f"API Error: {response.status} {response.reason}"
                        }]
                    }
                data = await response.json()
                return {
                    "content": [{
                        "type": "text",
                        "text": json.dumps(data, indent=2)
                    }]
                }
    except asyncio.TimeoutError:
        return {
            "content": [{
                "type": "text",
                "text": "Error: Request timed out after 10 seconds"
            }]
        }
    except Exception as e:
        return {
            "content": [{
                "type": "text",
                "text": f"Error: {type(e).__name__}: {str(e)}"
            }]
        }
```

## Real-World Tool Examples

### Database Query Tool

```python
import asyncpg

@tool(
    "query_database",
    "Execute a read-only SQL query",
    {"query": str, "params": list}
)
async def query_database(args: dict[str, Any]) -> dict[str, Any]:
    query = args["query"]
    params = args.get("params", [])

    # Validate read-only
    if not query.strip().upper().startswith("SELECT"):
        return {
            "content": [{
                "type": "text",
                "text": "Error: Only SELECT queries allowed"
            }]
        }

    try:
        conn = await asyncpg.connect(os.environ["DATABASE_URL"])
        try:
            rows = await conn.fetch(query, *params)
            result = [dict(row) for row in rows]
            return {
                "content": [{
                    "type": "text",
                    "text": json.dumps(result, indent=2, default=str)
                }]
            }
        finally:
            await conn.close()
    except Exception as e:
        return {
            "content": [{
                "type": "text",
                "text": f"Database error: {e}"
            }]
        }
```

### HTTP API Tool

```typescript
const apiRequestTool = tool(
  "api_request",
  "Make an HTTP API request",
  {
    url: z.string().url(),
    method: z.enum(["GET", "POST", "PUT", "DELETE"]).default("GET"),
    headers: z.record(z.string()).optional(),
    body: z.any().optional(),
    timeout: z.number().int().min(1000).max(30000).default(10000)
  },
  async (args) => {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), args.timeout);

    try {
      const response = await fetch(args.url, {
        method: args.method,
        headers: args.headers,
        body: args.body ? JSON.stringify(args.body) : undefined,
        signal: controller.signal
      });

      clearTimeout(timeoutId);

      const contentType = response.headers.get("content-type") ?? "";
      let data: string;

      if (contentType.includes("application/json")) {
        data = JSON.stringify(await response.json(), null, 2);
      } else {
        data = await response.text();
      }

      return {
        content: [{
          type: "text" as const,
          text: `Status: ${response.status}\n\n${data}`
        }]
      };
    } catch (error) {
      clearTimeout(timeoutId);
      return {
        content: [{
          type: "text" as const,
          text: `Request failed: ${(error as Error).message}`
        }]
      };
    }
  }
);
```

### File Processing Tool

```python
import base64

@tool(
    "process_image",
    "Process an image file (resize, convert, etc.)",
    {
        "file_path": str,
        "operation": str,  # "resize", "convert", "thumbnail"
        "options": dict
    }
)
async def process_image(args: dict[str, Any]) -> dict[str, Any]:
    from PIL import Image
    import io

    try:
        img = Image.open(args["file_path"])
        operation = args["operation"]
        options = args.get("options", {})

        if operation == "resize":
            width = options.get("width", img.width)
            height = options.get("height", img.height)
            img = img.resize((width, height))
        elif operation == "thumbnail":
            size = options.get("size", 128)
            img.thumbnail((size, size))
        elif operation == "convert":
            format = options.get("format", "PNG")
            # Convert handled on save

        # Save to bytes
        buffer = io.BytesIO()
        img.save(buffer, format=options.get("format", "PNG"))
        buffer.seek(0)

        # Return base64 encoded
        encoded = base64.b64encode(buffer.read()).decode()

        return {
            "content": [{
                "type": "text",
                "text": f"Processed image: {img.width}x{img.height}\nBase64 length: {len(encoded)}"
            }]
        }
    except Exception as e:
        return {
            "content": [{
                "type": "text",
                "text": f"Image processing error: {e}"
            }]
        }
```

## Multiple Tools in One Server

```python
from claude_agent_sdk import tool, create_sdk_mcp_server

@tool("list_users", "List all users", {})
async def list_users(args):
    users = await db.fetch_all("SELECT * FROM users")
    return {"content": [{"type": "text", "text": json.dumps(users)}]}

@tool("get_user", "Get user by ID", {"user_id": str})
async def get_user(args):
    user = await db.fetch_one("SELECT * FROM users WHERE id = $1", args["user_id"])
    return {"content": [{"type": "text", "text": json.dumps(user)}]}

@tool("create_user", "Create new user", {"name": str, "email": str})
async def create_user(args):
    user = await db.execute(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
        args["name"], args["email"]
    )
    return {"content": [{"type": "text", "text": json.dumps(user)}]}

server = create_sdk_mcp_server(
    name="users",
    version="1.0.0",
    tools=[list_users, get_user, create_user]
)

# All tools available:
# - mcp__users__list_users
# - mcp__users__get_user
# - mcp__users__create_user
```

## Registering with Agent

```python
options = ClaudeAgentOptions(
    mcp_servers={"users": server},
    allowed_tools=[
        "Read",  # Built-in tools
        "Write",
        "mcp__users__list_users",    # Custom tools
        "mcp__users__get_user",
        "mcp__users__create_user"
    ]
)
```

## External MCP Servers

For complex tools, use external MCP server processes:

```python
options = ClaudeAgentOptions(
    mcp_servers={
        "playwright": {
            "command": "npx",
            "args": ["@playwright/mcp@latest"]
        },
        "postgres": {
            "command": "npx",
            "args": ["@modelcontextprotocol/server-postgres"],
            "env": {"DATABASE_URL": os.environ["DATABASE_URL"]}
        },
        "github": {
            "command": "npx",
            "args": ["@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]}
        }
    }
)
```

## Best Practices

1. **Descriptive names**: Tool names should clearly indicate purpose
2. **Detailed descriptions**: Help Claude understand when to use the tool
3. **Parameter descriptions**: Use Zod `.describe()` or docstrings
4. **Error handling**: Always return graceful errors in content
5. **Timeouts**: Set appropriate timeouts for external calls
6. **Validation**: Validate inputs before processing
7. **Idempotency**: Design tools to be safely retried
8. **Least privilege**: Only expose necessary functionality
