---
name: mcp-builder:validate
description: Validate your MCP server against best practices and catch common anti-patterns
---

# Validate MCP Server

You are helping the user validate their MCP server against best practices.

## Validation Categories

1. **Tool Design** - Are tools following "true task" principles?
2. **Descriptions** - Do descriptions help agents know when to use tools?
3. **Input Schemas** - Are parameters well-defined with descriptions?
4. **Error Handling** - Are errors actionable for agents?
5. **Security** - Are there any obvious vulnerabilities?

## Workflow

### Phase 1: Locate Server Files

Find the MCP server implementation:
- Python: Look for `server.py` or files with `@mcp.tool()` decorators
- TypeScript: Look for `index.ts` with `ListToolsRequestSchema` handlers

### Phase 2: Run Validation Checks

#### Check 1: Tool Naming

**Rule:** Tool names should be kebab-case and describe the action.

```
PASS: get-stock-summary, search-tasks, create-user
FAIL: getStockSummary, get_stock, task, doThing
```

#### Check 2: Description Quality

**Rule:** Descriptions should tell the agent WHEN to use the tool.

```
PASS: "Search for tasks by keyword. Returns full task details including
       content, assignee, and due date. Use when the user wants to find
       specific tasks or see what's assigned to them."

FAIL: "Searches tasks"
FAIL: "This tool searches for tasks in the database"
```

**Validation criteria:**
- [ ] Describes what the tool does
- [ ] Explains when to use it
- [ ] Mentions what's returned

#### Check 3: Sequential Call Anti-Pattern

**Rule:** Tools should not require sequential calling.

```
FAIL: Tool A returns IDs, Tool B requires those IDs
      - find_users(query) -> [id1, id2]
      - get_user(id) -> {details}

PASS: Single tool returns complete result
      - search_users(query, include_details=true) -> [{full_user}]
```

**Detection:**
- Look for tools that return only IDs or references
- Look for tools that take IDs as their only input
- Check if tool outputs could be inputs to other tools

#### Check 4: Input Schema Completeness

**Rule:** All parameters should have descriptions.

```python
# FAIL - No descriptions
class Input(BaseModel):
    query: str
    limit: int

# PASS - Clear descriptions
class Input(BaseModel):
    query: str = Field(..., description="Search term for task subject or content")
    limit: int = Field(10, description="Maximum results to return (1-100)")
```

#### Check 5: Error Message Quality

**Rule:** Errors should be actionable for agents.

```
FAIL: "Error occurred"
FAIL: "Invalid input"
FAIL: "500 Internal Server Error"

PASS: "Task not found with ID: abc123. Please verify the task exists."
PASS: "Invalid date format. Expected ISO 8601 (e.g., 2025-01-15T10:30:00Z)"
PASS: "Rate limit exceeded. Try again in 60 seconds."
```

#### Check 6: Resource Definition

**Rule:** If tools need context, resources should provide it.

```
WARNING: Tool 'query_database' exists but no schema resource found.
         Consider adding a schema://database resource so agents
         understand the data structure.
```

#### Check 7: Security Review

**Rules:**
- [ ] No hardcoded secrets in code
- [ ] Sensitive tools require authentication
- [ ] SQL queries use parameterized statements
- [ ] User input is validated before use

### Phase 3: Generate Report

Output a validation report:

```markdown
# MCP Server Validation Report

## Summary
- Tools: 5
- Resources: 2
- Issues Found: 3

## Tool Analysis

### get-stock-summary
Status: PASS
- Name: kebab-case
- Description: Explains when to use
- Schema: All fields documented

### search-tasks
Status: WARNING
- Name: kebab-case
- Description: Could be more specific about return format
- Schema: Missing description for 'status' parameter

### get-task-details
Status: FAIL
- Sequential Call Pattern: This tool takes an ID that comes from
  'search-tasks'. Consider merging into search-tasks with
  include_details parameter.

## Recommendations

1. Merge get-task-details into search-tasks
2. Add description to 'status' parameter in search-tasks
3. Consider adding schema://database resource for context

## Security Notes

- No hardcoded secrets found
- Database queries use parameterized statements
- Consider adding authentication for write operations
```

### Phase 4: Provide Fixes

For each issue, provide specific fix suggestions:

```python
# ISSUE: Sequential call pattern
# BEFORE:
@mcp.tool()
def search_tasks(query: str) -> List[str]:
    """Search for task IDs matching query."""
    return [t.id for t in db.search(query)]

@mcp.tool()
def get_task(id: str) -> dict:
    """Get task details by ID."""
    return db.get_task(id)

# AFTER:
@mcp.tool()
def search_tasks(
    query: str,
    include_details: bool = True
) -> List[dict]:
    """Search for tasks matching query. Returns full task details
    including content, status, and assignee."""
    tasks = db.search(query)
    if include_details:
        return [task.to_dict() for task in tasks]
    return [{"id": t.id, "subject": t.subject} for t in tasks]
```

## Quick Validation Commands

**Python:**
```bash
# Check syntax
python -m py_compile server.py

# Type check (if using mypy)
mypy server.py

# Run server and check tool listing
python server.py &
curl http://localhost:8000/mcp -d '{"method":"tools/list"}'
```

**TypeScript:**
```bash
# Type check
tsc --noEmit

# Run and test
npm run dev &
curl http://localhost:3000/mcp -d '{"method":"tools/list"}'
```
