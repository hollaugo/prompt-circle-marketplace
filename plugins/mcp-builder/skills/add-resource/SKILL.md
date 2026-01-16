---
name: mcp-builder:add-resource
description: Add a resource to your MCP server for agent context (schemas, templates, reference data)
---

# Add MCP Resource

You are helping the user add a resource to their MCP server. Resources provide context that agents can read before calling tools.

## What Are Resources?

Resources are data the agent can READ to understand context:
- **Database schemas** - So the agent knows table structure
- **API documentation** - So the agent knows available endpoints
- **Templates** - Pre-built formats for common outputs
- **Reference data** - Static data the agent might need

## When to Use Resources vs Tools

| Use Case | Resource or Tool? |
|----------|-------------------|
| Agent needs to read a schema before querying | **Resource** |
| Agent needs to fetch live data | **Tool** |
| Agent needs reference documentation | **Resource** |
| Agent needs to perform an action | **Tool** |

## Workflow

### Phase 1: Identify the Need

Ask the user:
1. **What context does the agent need?**
2. **Is this data static or dynamic?**
3. **Will this help the agent make better tool calls?**

### Phase 2: Design the Resource

**Common Resource Types:**

| Type | URI Pattern | Example |
|------|-------------|---------|
| Schema | `schema://{name}` | `schema://database` |
| Template | `template://{name}` | `template://report` |
| Reference | `ref://{category}/{name}` | `ref://api/endpoints` |
| Config | `config://{name}` | `config://settings` |

### Phase 3: Implement the Resource

**Python (FastMCP):**

```python
# Static resource
@mcp.resource("schema://database")
def get_database_schema() -> str:
    """Database schema for the task management system.
    Read this before querying to understand table structure."""
    return """
    Tables:
    - tasks (id, subject, content_md, status, due_at, assignee_id)
    - users (id, email, name, role)
    - comments (id, task_id, user_id, content, created_at)

    Relationships:
    - tasks.assignee_id -> users.id
    - comments.task_id -> tasks.id
    - comments.user_id -> users.id
    """

# Dynamic resource with template
@mcp.resource("template://task-report")
def get_task_report_template() -> str:
    """Template for generating task status reports."""
    return """
    # Task Report: {title}

    ## Summary
    - Total Tasks: {total}
    - Completed: {completed}
    - In Progress: {in_progress}
    - Pending: {pending}

    ## Details
    {task_list}
    """
```

**TypeScript (MCP SDK):**

```typescript
server.setRequestHandler(ListResourcesRequestSchema, async () => ({
  resources: [
    {
      uri: "schema://database",
      name: "Database Schema",
      description: "Task management database structure",
      mimeType: "text/plain",
    },
  ],
}));

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;

  if (uri === "schema://database") {
    return {
      contents: [{
        uri,
        mimeType: "text/plain",
        text: `Tables:\n- tasks (id, subject, status...)\n- users (id, email...)`,
      }],
    };
  }

  throw new Error(`Unknown resource: ${uri}`);
});
```

### Phase 4: Document the Resource

Add a clear description:

```python
@mcp.resource("schema://database")
def get_database_schema() -> str:
    """Database schema for the task management system.

    Read this resource before:
    - Building database queries
    - Understanding data relationships
    - Validating user input

    The schema includes tables, columns, types, and relationships.
    """
```

## Resource vs Tool: Database Example

**Resource (Schema):**
```python
@mcp.resource("schema://database")
def get_schema() -> str:
    """Returns the database schema for reference."""
    return "Tables: tasks, users, comments..."
```

**Tool (Query):**
```python
@mcp.tool()
def query_database(sql: str) -> str:
    """Execute a read-only SQL query against the database."""
    return db.execute(sql)
```

**Agent Workflow:**
1. Agent reads `schema://database` resource
2. Agent understands table structure
3. Agent constructs appropriate query
4. Agent calls `query_database` tool

## Best Practices

1. **Keep resources focused** - One resource per concern
2. **Use clear URIs** - `schema://`, `template://`, `ref://`
3. **Document when to read** - Help agent know when this resource is useful
4. **Keep content concise** - Agents have limited context

## Update State

After adding the resource, update `.mcp-builder/state.json`:

```json
{
  "resources": [
    { "uri": "schema://database", "name": "Database Schema" }
  ]
}
```
