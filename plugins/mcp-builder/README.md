# MCP Builder Plugin

Build production-ready MCP (Model Context Protocol) servers with AI guidance and best practices.

## Philosophy

> "MCP servers should NOT be pure API wrappers. Design tools that satisfy user intent in a single call."

This plugin teaches you to think like a server designer: empathize with the agent (your client), identify true tasks, and build tools that produce complete results.

## Installation

```bash
# Add the Prompt Circle marketplace
claude /plugin add hollaugo/prompt-circle-marketplace

# Install the MCP Builder plugin
claude /plugin install mcp-builder
```

## Quick Start

```bash
# Create a new MCP server
/mcp-builder:new-mcp

# Add a tool to your server
/mcp-builder:add-tool

# Validate against best practices
/mcp-builder:validate
```

## Available Skills

| Command | Description |
|---------|-------------|
| `/mcp-builder:new-mcp` | Scaffold a new MCP server with your chosen pattern |
| `/mcp-builder:add-tool` | Add a tool following "true task" design principles |
| `/mcp-builder:add-resource` | Add a resource for agent context |
| `/mcp-builder:add-auth` | Add Auth0/OAuth authentication |
| `/mcp-builder:validate` | Check server against best practices |
| `/mcp-builder:test` | Test with MCP Inspector |
| `/mcp-builder:deploy` | Generate deployment config (Cloud Run, Render, Fly.io) |

## Specialized Agents

| Agent | Purpose |
|-------|---------|
| `mcp-architect` | Server structure, tool identification, data flow design |
| `tool-designer` | Input schemas, descriptions, error handling |
| `schema-validator` | Best practices checking, anti-pattern detection |
| `deploy-orchestrator` | Container config, cloud deployment |

## Server Templates

| Pattern | Best For |
|---------|----------|
| **Basic** | Simple tools, learning, quick prototypes |
| **Tool-Heavy** | Data analysis, many tools, financial/research apps |
| **Widget-Enabled** | React UI output, ChatGPT Apps, rich visualizations |
| **Database-Backed** | Authentication, user scoping, state management |

## Core Principles

### 1. Empathize with the Client
Your client is an AI agent (Claude, ChatGPT, Cursor). Design for how agents think.

### 2. True Tasks, Not API Wrappers
**Bad:** `find_task(query)` returns IDs → `get_task(id)` returns details
**Good:** `search_tasks(query, include_details=true)` → complete result

### 3. Self-Describing Tools
Tool names and descriptions tell the agent WHEN to use it, not just what it does.

### 4. Atomic but Complete
One tool, one job, full result. No sequential calling required.

### 5. Predictable Responses
Same inputs always produce same output shape. Agents can reason about structure.

## Tool Design Checklist

- [ ] Tool name clearly describes its purpose
- [ ] Description tells agent WHEN to use it
- [ ] Input schema has descriptions for all parameters
- [ ] Output is complete (agent doesn't need another call)
- [ ] Error messages are actionable

## Learn More

This plugin is part of the [MCP Server Mastery](https://promptcircle.ai/courses/mcp-server-mastery) course.

## Support

- Documentation: https://promptcircle.ai/docs/mcp-builder
- Issues: https://github.com/hollaugo/prompt-circle-marketplace/issues
- Email: support@promptcircle.ai
