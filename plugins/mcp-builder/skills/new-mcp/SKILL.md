---
name: mcp-builder:new-mcp
description: Scaffold a new MCP server with your chosen pattern (basic, tool-heavy, widget-enabled, or database-backed)
---

# Create New MCP Server

You are helping the user create a new MCP (Model Context Protocol) server from scratch.

## Philosophy

> "This is not a code walkthrough. The true value lies in the problem you're solving, how you think about solving it, and empathizing with the client (the agent consuming your server)."

## Workflow

### Phase 1: Understand the Problem

Ask the user:
1. **What problem are you solving?** What will the agent help users accomplish?
2. **Who is the end user?** (e.g., financial advisors, developers, healthcare workers)
3. **What data/systems will the agent access?** (APIs, databases, files)

### Phase 2: Choose Architecture Pattern

Present these options based on their needs:

| Pattern | Best For | Complexity |
|---------|----------|------------|
| **Basic** | Learning, simple tools, quick prototypes | Low |
| **Tool-Heavy** | Data analysis, many tools (like financial research) | Medium |
| **Widget-Enabled** | Rich UI output, ChatGPT Apps, visualizations | Medium-High |
| **Database-Backed** | User auth, state persistence, multi-tenant | High |

### Phase 3: Identify True Tasks

Guide the user to think about user intents, NOT API endpoints:

```
User Intent → Outcome → Tool Design
"What's AAPL's price?" → Complete stock summary → get_stock_summary(ticker)
"Compare these stocks" → Side-by-side analysis → compare_stocks(tickers[])
```

**Anti-pattern to avoid:**
- Creating `get_task_ids()` + `get_task_details()` when you could have `search_tasks(include_details=true)`

### Phase 4: Define Initial Tools

For each tool, capture:
- **Name** (kebab-case): `get-stock-summary`, `search-tasks`
- **Description**: When should the agent use this? (not just what it does)
- **Inputs**: What parameters? Which are required?
- **Output**: What does a complete result look like?

### Phase 5: Generate Server

Based on the chosen pattern, use the `mcp-builder:mcp-architect` agent to generate:

**For Python (FastMCP):**
```
{server-name}/
├── server.py              # FastMCP server with tools
├── requirements.txt       # Python dependencies
├── Dockerfile            # Container config
├── .env.example          # Environment template
└── src/                  # Business logic (if tool-heavy)
    └── {domain}.py       # Domain-specific modules
```

**For TypeScript (MCP SDK):**
```
{server-name}/
├── server/
│   └── index.ts          # MCP server
├── package.json
├── tsconfig.json
├── Dockerfile
└── .env.example
```

### Phase 6: Verify Setup

After generation:
1. Run `pip install -r requirements.txt` or `npm install`
2. Start the server
3. Test with MCP Inspector: `npx @anthropic-ai/mcp-inspector`

## Server Naming

Use kebab-case: `stock-analysis-server`, `patient-records-server`, `task-manager-server`

## Language Selection

**Python (Recommended for most cases):**
- FastMCP makes tool creation simple
- Great for data processing, API integrations
- Most MCP examples are in Python

**TypeScript (For widget-enabled servers):**
- Required for ChatGPT Apps with inline widgets
- Better if you're building React UI components

## State Management

Create `.mcp-builder/state.json` to track progress:

```json
{
  "serverName": "stock-analysis-server",
  "pattern": "tool-heavy",
  "language": "python",
  "tools": [
    { "name": "get-stock-summary", "status": "implemented" }
  ],
  "resources": [],
  "createdAt": "2025-01-15T00:00:00Z"
}
```

## Next Steps

After scaffolding, suggest:
- `/mcp-builder:add-tool` - Add more tools
- `/mcp-builder:validate` - Check best practices
- `/mcp-builder:test` - Test with MCP Inspector
