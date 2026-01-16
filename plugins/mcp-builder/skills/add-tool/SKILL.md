---
name: mcp-builder:add-tool
description: Add a new tool to your MCP server using "true task" design principles
---

# Add MCP Tool

You are helping the user add a new tool to their MCP server using best practices.

## Philosophy

> "MCP servers should NOT be pure API wrappers. Design tools that satisfy user intent in a single call."

## Workflow

### Phase 1: Understand the Intent

Ask the user:
1. **What will someone ask the agent?** (User intent)
2. **What result satisfies that intent?** (Desired outcome)
3. **Is there existing code/API to leverage?**

### Phase 2: Design the Tool

Apply the "True Task" framework:

```
User Intent → Outcome → Tool Name
```

**Good Examples:**
| User Intent | Outcome | Tool |
|-------------|---------|------|
| "What's AAPL trading at?" | Complete stock summary | `get_stock_summary(ticker)` |
| "Find tasks about the homepage" | Full task details | `search_tasks(query, include_details=true)` |
| "Compare MSFT and GOOGL" | Side-by-side analysis | `compare_stocks(tickers[])` |

**Bad Examples (API Wrapper Anti-pattern):**
| Bad Design | Problem |
|------------|---------|
| `get_task_ids(query)` then `get_task(id)` | Agent must make 2 calls |
| `list_all_records()` then filter | Too much data, agent must process |
| `search()` returns IDs only | Incomplete result |

### Phase 3: Define Input Schema

Create a schema with:
- **Required fields**: What's absolutely needed?
- **Optional fields**: What provides flexibility?
- **Descriptions**: Help the agent understand each field

**Python (Pydantic):**
```python
class SearchTasksInput(BaseModel):
    query: str = Field(..., description="Search term for task subject or content")
    include_details: bool = Field(True, description="Include full task content")
    status: Optional[str] = Field(None, description="Filter by status: pending, in_progress, done")
    limit: int = Field(10, description="Max results (1-50)")

    model_config = ConfigDict(extra="forbid")
```

**TypeScript (Zod):**
```typescript
const SearchTasksSchema = z.object({
  query: z.string().describe("Search term for task subject or content"),
  include_details: z.boolean().default(true).describe("Include full task content"),
  status: z.enum(["pending", "in_progress", "done"]).optional(),
  limit: z.number().min(1).max(50).default(10),
});
```

### Phase 4: Write the Description

The description tells the agent WHEN to use this tool:

**Good:** "Search for tasks by keyword. Returns full task details including content, assignee, and due date. Use when the user wants to find specific tasks or see what's assigned to them."

**Bad:** "Searches tasks" (too vague, doesn't explain when to use)

### Phase 5: Implement the Tool

**Python (FastMCP):**
```python
@mcp.tool()
async def search_tasks(
    query: str,
    include_details: bool = True,
    status: Optional[str] = None,
    limit: int = 10
) -> str:
    """Search for tasks by keyword. Returns full task details including
    content, assignee, and due date. Use when the user wants to find
    specific tasks or see what's assigned to them."""

    # Validate input
    payload = SearchTasksInput(
        query=query,
        include_details=include_details,
        status=status,
        limit=limit
    )

    # Business logic
    results = await task_service.search(payload)

    # Return complete result
    return json.dumps(results, indent=2)
```

### Phase 6: Add Error Handling

Return actionable error messages:

```python
try:
    results = await task_service.search(payload)
    return json.dumps(results)
except ValidationError as e:
    return f"Invalid input: {e.errors()}"
except PermissionError:
    return "Access denied. User does not have permission to search tasks."
except Exception as e:
    return f"Search failed: {str(e)}. Please try again or refine your query."
```

## Tool Design Checklist

Before finalizing, verify:

- [ ] **Name** clearly describes purpose (kebab-case)
- [ ] **Description** tells agent WHEN to use it
- [ ] **Inputs** have descriptions for all parameters
- [ ] **Output** is complete (no follow-up call needed)
- [ ] **Errors** are actionable (agent can reason about them)

## Tool Annotations (for ChatGPT Apps)

Set appropriate hints:

| Scenario | readOnlyHint | destructiveHint | openWorldHint |
|----------|--------------|-----------------|---------------|
| List/Get/Search | `true` | `false` | `false` |
| Create/Update | `false` | `false` | `false` |
| Delete | `false` | `true` | `false` |
| External API call | varies | varies | `true` |

## Update State

After adding the tool, update `.mcp-builder/state.json`:

```json
{
  "tools": [
    { "name": "search-tasks", "status": "implemented", "type": "query" }
  ]
}
```

## Next Steps

- `/mcp-builder:validate` - Check tool follows best practices
- `/mcp-builder:test` - Test with MCP Inspector
