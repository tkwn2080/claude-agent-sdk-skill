# claude-agent-sdk-skill

A Claude Code skill for building production AI agents with the [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview).

## Installation

### Option 1: Plugin Marketplace (Recommended)

In Claude Code, run:

```
/plugin marketplace add tkwn2080/claude-agent-sdk-skill
```

The skill will be automatically available after adding the marketplace.

### Option 2: Manual Installation

Clone and copy to your personal skills directory:

```bash
git clone https://github.com/tkwn2080/claude-agent-sdk-skill.git
cp -r claude-agent-sdk-skill/skills/claude-agent-sdk ~/.claude/skills/
```

Then restart Claude Code.

## What This Skill Provides

When working with the Claude Agent SDK, Claude will automatically:

- Use correct `query()` and `ClaudeSDKClient` patterns
- Apply proper async/await patterns for Python and TypeScript
- Create custom tools with correct MCP naming (`mcp__{server}__{tool}`)
- Implement hooks with proper permission decision flow (deny > ask > allow)
- Design multi-agent systems using orchestrator-worker patterns
- Handle sessions and context resumption correctly
- Follow security best practices (deny-by-default, confirmation gates)

## Coverage

- **Core APIs**: `query()` for one-off tasks, `ClaudeSDKClient` for conversations
- **Built-in Tools**: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, Task
- **Custom Tools**: In-process MCP servers, `@tool` decorator, `createSdkMcpServer`
- **Hooks System**: PreToolUse, PostToolUse, Stop, SubagentStop lifecycle hooks
- **Subagents**: Parallel task delegation with specialized toolsets
- **Multi-Agent**: Orchestrator-worker, routing, parallelization patterns
- **Sessions**: Context resumption across queries
- **Security**: Permission models, secret management, audit logging

## Structure

```
claude-agent-sdk-skill/
├── .claude-plugin/
│   └── plugin.json                    # Plugin metadata
├── skills/
│   └── claude-agent-sdk/
│       ├── SKILL.md                   # Main skill entry point
│       └── references/
│           ├── python-patterns.md     # Python examples and patterns
│           ├── typescript-patterns.md # TypeScript examples and patterns
│           ├── hooks-reference.md     # Complete hooks documentation
│           ├── custom-tools.md        # Tool creation guide
│           └── multi-agent.md         # Orchestration patterns
├── README.md
└── LICENSE
```

## Requirements

- Claude Code CLI
- Anthropic API key (or Bedrock/Vertex credentials)
- Python 3.10+ (for Python SDK)
- Node.js 18+ (for TypeScript SDK)

## Sources

This skill is based on primary Anthropic documentation:

- [Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Building Agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [How We Built Our Multi-Agent Research System](https://www.anthropic.com/engineering/multi-agent-research-system)
- [GitHub: anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [GitHub: anthropics/claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)

## License

MIT
