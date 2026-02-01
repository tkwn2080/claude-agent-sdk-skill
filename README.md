# Claude Agent SDK Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for building production AI agents with the [Claude Agent SDK](https://docs.anthropic.com/en/docs/agents-and-tools/claude-agent-sdk).

## Install

**Plugin (recommended):**
```
/install-plugin github:tkwn2080/claude-agent-sdk-skill
```

**Manual:**
```bash
git clone https://github.com/tkwn2080/claude-agent-sdk-skill.git
cp -r claude-agent-sdk-skill/skills/claude-agent-sdk ~/.claude/skills/
```

Restart Claude Code after installing.

## What it does

When you work with the Agent SDK, Claude automatically applies correct patterns for:

- `query()` and `ClaudeSDKClient` usage
- Custom MCP tools (`mcp__{server}__{tool}` naming, streaming input)
- Hooks and permission decisions (deny > ask > allow)
- Multi-agent orchestration (orchestrator-worker, routing, parallelization)
- Session management and context resumption
- Security (deny-by-default, confirmation gates, audit logging)

## Coverage

| Topic | File | What's covered |
|-------|------|----------------|
| Core APIs | `SKILL.md` | `query()`, `ClaudeSDKClient`, configuration options, authentication (incl. OpenRouter) |
| Python | `references/python-patterns.md` | All patterns: streaming input, hooks, permissions, `can_use_tool`, error handling |
| TypeScript | `references/typescript-patterns.md` | All patterns: Zod schemas, hooks, `canUseTool`, Query object methods |
| Hooks | `references/hooks-reference.md` | All 12 hook events, permission evaluation order, `acceptEdits` scope, troubleshooting |
| Custom tools | `references/custom-tools.md` | `@tool` decorator, `createSdkMcpServer`, external MCP servers, annotations |
| Multi-agent | `references/multi-agent.md` | Orchestrator-worker, routing, parallelization, evaluator-optimizer, prompt chaining |
| Known issues | Across all files | Stream closed fixes, MCP + subagent limitations, `can_use_tool` bugs, headless mode |

## Structure

```
skills/claude-agent-sdk/
├── SKILL.md                          # Entry point — quick start, config, known issues
└── references/
    ├── python-patterns.md            # Python SDK patterns and examples
    ├── typescript-patterns.md        # TypeScript SDK patterns and examples
    ├── hooks-reference.md            # Hooks, permissions, troubleshooting
    ├── custom-tools.md               # Tool creation and MCP servers
    └── multi-agent.md                # Multi-agent orchestration patterns
```

## Requirements

- Claude Code CLI
- Anthropic API key (or Bedrock / Vertex / Azure / OpenRouter credentials)
- Python 3.10+ or Node.js 18+

## Sources

- [Agent SDK docs](https://docs.anthropic.com/en/docs/agents-and-tools/claude-agent-sdk)
- [Building Agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [anthropics/claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)

## License

MIT
