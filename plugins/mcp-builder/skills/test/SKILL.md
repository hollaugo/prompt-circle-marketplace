---
name: mcp-builder:test
description: Test your MCP server with MCP Inspector and verify tool functionality
---

# Test MCP Server

You are helping the user test their MCP server to ensure it works correctly.

## Testing Tools

1. **MCP Inspector** - Interactive GUI for testing MCP servers
2. **curl** - Command-line HTTP testing
3. **Integration tests** - Automated test scripts

## Workflow

### Phase 1: Start the Server

Use the START.sh script if available:

```bash
# Development mode (auto-reload)
./START.sh --dev

# Or start manually:
cd your-server-directory
python server.py
# Server running at http://localhost:8000
```

### Phase 2: Test with MCP Inspector

The MCP Inspector is the recommended way to test your server interactively.

**For Streamable HTTP servers:**
```bash
# Start your server first, then run Inspector separately
./START.sh --dev &
npx @modelcontextprotocol/inspector
```

Then in the Inspector UI:
1. Select "Streamable HTTP" transport
2. Enter server URL: `http://localhost:8000/mcp`
3. Click "Connect"

**For stdio servers:**
```bash
# Inspector launches and manages your server
npx @modelcontextprotocol/inspector python server.py
```

**For Python servers with uv:**
```bash
npx @modelcontextprotocol/inspector uv --directory . run your-server
```

**In the Inspector UI:**
1. View available tools in the Tools tab
2. Test individual tools with sample inputs
3. View resources in the Resources tab
4. Check prompts in the Prompts tab
5. Monitor logs in the Notifications pane

### Phase 3: Test Tools Manually

**List available tools:**
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

**Call a tool:**
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0",
    "id":2,
    "method":"tools/call",
    "params":{
      "name":"get-stock-summary",
      "arguments":{"ticker":"AAPL"}
    }
  }'
```

**List resources:**
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"resources/list"}'
```

**Read a resource:**
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0",
    "id":4,
    "method":"resources/read",
    "params":{"uri":"schema://database"}
  }'
```

### Phase 4: Create Test Cases

For each tool, create test cases:

```python
# tests/test_tools.py
import pytest
import httpx

BASE_URL = "http://localhost:8000/mcp"

async def call_tool(name: str, arguments: dict) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(BASE_URL, json={
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tools/call",
            "params": {"name": name, "arguments": arguments}
        })
        return response.json()

class TestStockTools:
    @pytest.mark.asyncio
    async def test_get_stock_summary_valid_ticker(self):
        result = await call_tool("get-stock-summary", {"ticker": "AAPL"})
        assert "error" not in result
        assert "content" in result.get("result", {})

    @pytest.mark.asyncio
    async def test_get_stock_summary_invalid_ticker(self):
        result = await call_tool("get-stock-summary", {"ticker": "INVALID123"})
        # Should return error, not crash
        content = result.get("result", {}).get("content", [])
        assert any("error" in str(c).lower() or "not found" in str(c).lower()
                   for c in content)

    @pytest.mark.asyncio
    async def test_compare_stocks_multiple(self):
        result = await call_tool("compare-stocks", {
            "tickers": ["AAPL", "MSFT", "GOOGL"]
        })
        assert "error" not in result
```

### Phase 5: Test Edge Cases

**Test these scenarios for each tool:**

| Scenario | What to Check |
|----------|---------------|
| Valid input | Returns expected result |
| Missing required field | Returns clear error message |
| Invalid data type | Returns validation error |
| Empty input | Handles gracefully |
| Very large input | Doesn't timeout or crash |
| Special characters | Handles encoding properly |
| Null/undefined values | Doesn't crash |

### Phase 6: Test with Real Clients

**Claude Desktop:**
1. Add server to `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "your-server": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```
2. Restart Claude Desktop
3. Test natural language queries

**Cursor:**
1. Add to Cursor MCP settings
2. Test tool invocations from chat

### Phase 7: Performance Testing

**Measure response times:**
```bash
# Simple timing
time curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search-tasks","arguments":{"query":"test"}}}'

# Load testing with hey
hey -n 100 -c 10 -m POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  http://localhost:8000/mcp
```

**Performance targets:**
- Tool listing: < 100ms
- Simple queries: < 500ms
- Complex operations: < 2s

## Test Checklist

Before deployment, verify:

- [ ] All tools listed correctly
- [ ] All resources accessible
- [ ] Valid inputs return expected results
- [ ] Invalid inputs return helpful errors
- [ ] Edge cases handled gracefully
- [ ] Performance meets targets
- [ ] Works with at least one MCP client

## Common Issues

**"Connection refused":**
- Server not running
- Wrong port
- Firewall blocking

**"Method not found":**
- Tool name typo
- Tool not registered

**"Invalid params":**
- Missing required fields
- Wrong data types

**"Timeout":**
- Server processing too slow
- Infinite loop in handler
- External API not responding
