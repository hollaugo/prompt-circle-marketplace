---
name: mcp-builder:new-mcp-app
description: Scaffold a complete MCP App with visual UI tools (TypeScript + React + Tailwind)
---

# Create New MCP App

You are helping the user create a new MCP App - an MCP server with visual UI components that render inline in ChatGPT or Claude.

## Critical Dependencies (MUST USE EXACT VERSIONS)

These packages are required - do NOT substitute or use different versions:

```json
{
  "dependencies": {
    "@modelcontextprotocol/ext-apps": "^1.0.0",
    "@modelcontextprotocol/sdk": "^1.24.0",
    "zod": "^4.1.13",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "cors": "^2.8.5",
    "express": "^5.1.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.4",
    "vite": "^6.0.0",
    "vite-plugin-singlefile": "^2.3.0",
    "tailwindcss": "^3.4.17",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.49",
    "tsx": "^4.19.0",
    "typescript": "^5.9.3"
  }
}
```

## Key Differences from Standard MCP Servers

| Aspect | Standard MCP Server | MCP App |
|--------|---------------------|---------|
| UI Output | Text/JSON only | Visual React components |
| Package | `@modelcontextprotocol/sdk` | + `@modelcontextprotocol/ext-apps` |
| Schema | JSON Schema objects | **Zod schemas directly** |
| Tool Registration | `server.tool()` | `registerAppTool()` |
| Resource Registration | `server.resource()` | `registerAppResource()` |
| Server Pattern | Single instance | `createServer()` factory per request |
| HTTP Handler | `app.post("/mcp", ...)` | `app.all("/mcp", ...)` |
| Express Setup | Manual | `createMcpExpressApp()` helper |

## Workflow

### Phase 1: Understand the App

Ask the user:
1. **What data will this visualize?** (e.g., stock prices, tasks, metrics, analytics)
2. **What UI pattern?** (Card, Chart, Table, Dashboard, Form)
3. **What API/data source?** (REST API, database, generated data)
4. **How many tools?** (Start with 1-2, can add more later)

### Phase 2: Design Tools with UIs

Each tool needs:
- **Tool definition** with zod schema inputs
- **Visual component** (React) for rendering results
- **Resource URI** linking tool to UI

Example mapping:
```
Tool: get-stock-detail
  -> inputSchema: { symbol: z.string() }
  -> resourceUri: "ui://stock-detail/app.html"
  -> Component: StockDetailCard.tsx
```

### Phase 3: Generate Project Structure

```
{app-name}/
├── server.ts              # MCP server with tools + resources
├── package.json           # Dependencies (exact versions critical)
├── vite.config.ts         # Single-file bundling config
├── tsconfig.json          # TypeScript config
├── tailwind.config.js     # Tailwind with host CSS vars
├── postcss.config.js      # PostCSS for Tailwind
├── .gitignore             # Ignore node_modules, dist
├── src/
│   ├── index.css          # Global styles + host CSS vars
│   ├── {tool-name}.tsx    # React app for each tool
│   └── components/        # Reusable UI components
└── {tool-name}.html       # Entry HTML for each tool UI
```

### Phase 4: Server Implementation

The server MUST follow this exact pattern:

```typescript
import {
  registerAppTool,
  registerAppResource,
  RESOURCE_MIME_TYPE,
} from "@modelcontextprotocol/ext-apps/server";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { createMcpExpressApp } from "@modelcontextprotocol/sdk/server/express.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import type { CallToolResult, ReadResourceResult } from "@modelcontextprotocol/sdk/types.js";
import cors from "cors";
import fs from "node:fs/promises";
import path from "node:path";
import { z } from "zod";

// Works both from source (server.ts) and compiled (dist/server.js)
const DIST_DIR = import.meta.filename.endsWith(".ts")
  ? path.join(import.meta.dirname, "dist")
  : import.meta.dirname;

// Resource URIs
const toolUIResourceUri = "ui://tool-name/app.html";

// Server Factory - CRITICAL: New server per request
export function createServer(): McpServer {
  const server = new McpServer({
    name: "App Name",
    version: "1.0.0",
  });

  // Register tool with zod schema (NOT JSON Schema)
  registerAppTool(
    server,
    "tool-name",
    {
      title: "Tool Title",
      description: "When to use this tool...",
      inputSchema: {
        param: z.string().describe("Parameter description"),
      },
      _meta: { ui: { resourceUri: toolUIResourceUri } },
    },
    async ({ param }): Promise<CallToolResult> => {
      const result = await fetchData(param);
      return {
        content: [{ type: "text", text: JSON.stringify(result) }],
        structuredContent: result,
      };
    }
  );

  // Register UI resource
  registerAppResource(
    server,
    toolUIResourceUri,
    toolUIResourceUri,
    { mimeType: RESOURCE_MIME_TYPE },
    async (): Promise<ReadResourceResult> => {
      const html = await fs.readFile(
        path.join(DIST_DIR, "tool-name", "tool-name.html"),
        "utf-8"
      );
      return {
        contents: [{ uri: toolUIResourceUri, mimeType: RESOURCE_MIME_TYPE, text: html }],
      };
    }
  );

  return server;
}

// HTTP Server - MUST use createMcpExpressApp and app.all
const port = parseInt(process.env.PORT ?? "3001", 10);
const app = createMcpExpressApp({ host: "0.0.0.0" });
app.use(cors());

app.all("/mcp", async (req, res) => {
  const server = createServer();
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,
  });

  res.on("close", () => {
    transport.close().catch(() => {});
    server.close().catch(() => {});
  });

  try {
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  } catch (error) {
    console.error("MCP error:", error);
    if (!res.headersSent) {
      res.status(500).json({
        jsonrpc: "2.0",
        error: { code: -32603, message: "Internal server error" },
        id: null,
      });
    }
  }
});

app.listen(port, () => {
  console.log(`Server listening on http://localhost:${port}/mcp`);
});
```

### Phase 5: React UI Implementation

Each tool UI follows this pattern:

```tsx
import { StrictMode, useState, useEffect } from "react";
import { createRoot } from "react-dom/client";
import { useApp, useHostStyles } from "@modelcontextprotocol/ext-apps/react";
import "./index.css";

interface ToolData {
  // Define based on tool output
}

function ToolUI() {
  const [data, setData] = useState<ToolData | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  const { app } = useApp({
    appInfo: { name: "Tool Name", version: "1.0.0" },
    onAppCreated: (app) => {
      app.ontoolresult = (result) => {
        setLoading(false);
        const text = result.content?.find((c) => c.type === "text")?.text;
        if (text) {
          try {
            const parsed = JSON.parse(text);
            if (parsed.error) {
              setError(parsed.message);
            } else {
              setData(parsed);
            }
          } catch {
            setError("Failed to parse data");
          }
        }
      };
    },
  });

  // Apply host CSS variables for theme integration
  useHostStyles(app);

  // Handle safe area insets for mobile
  useEffect(() => {
    if (!app) return;
    app.onhostcontextchanged = (ctx) => {
      if (ctx.safeAreaInsets) {
        const { top, right, bottom, left } = ctx.safeAreaInsets;
        document.body.style.padding = `${top}px ${right}px ${bottom}px ${left}px`;
      }
    };
  }, [app]);

  if (loading) return <LoadingSkeleton />;
  if (error) return <ErrorDisplay message={error} />;
  if (!data) return <EmptyState />;

  return <DataVisualization data={data} />;
}

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <ToolUI />
  </StrictMode>
);
```

### Phase 6: Host CSS Variable Integration

The CSS MUST use host CSS variables for theme compatibility:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  margin: 0;
  padding: 0;
  font-family: var(--font-sans, system-ui, -apple-system, sans-serif);
  background: var(--color-background-primary, #ffffff);
  color: var(--color-text-primary, #1a1a1a);
}

.card {
  background: var(--color-background-primary, #ffffff);
  border: 1px solid var(--color-border-primary, #e5e5e5);
  border-radius: var(--border-radius-lg, 12px);
  padding: 1.5rem;
}

.card-inner {
  background: var(--color-background-secondary, #f5f5f5);
  border-radius: var(--border-radius-md, 8px);
  padding: 1rem;
}

.text-secondary {
  color: var(--color-text-secondary, #525252);
}

.text-tertiary {
  color: var(--color-text-tertiary, #737373);
}
```

### Phase 7: Vite Single-File Bundling

The vite config MUST use the singlefile plugin:

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { viteSingleFile } from "vite-plugin-singlefile";

export default defineConfig({
  plugins: [react(), viteSingleFile()],
  build: {
    outDir: "dist",
    rollupOptions: {
      input: process.env.INPUT,
    },
  },
});
```

Build scripts (one entry point per tool):
```json
{
  "scripts": {
    "build": "npm run build:tool1 && npm run build:tool2",
    "build:tool1": "INPUT=tool1.html vite build --outDir dist/tool1",
    "build:tool2": "INPUT=tool2.html vite build --outDir dist/tool2",
    "serve": "tsx server.ts",
    "dev": "npm run build && npm run serve"
  }
}
```

## Common Gotchas

1. **zod@^4.1.13 required** - Older versions cause `v3Schema.safeParseAsync is not a function` error
2. **inputSchema uses zod directly** - NOT `{ type: "object", properties: {...} }`
3. **Handler args are destructured** - `async ({ symbol })` not `async (args)`
4. **app.all not app.post** - Required for all HTTP methods
5. **createServer() factory** - New server instance per request
6. **createMcpExpressApp helper** - Use SDK helper, not manual Express setup
7. **viteSingleFile plugin** - UI must be bundled into single HTML file
8. **RESOURCE_MIME_TYPE constant** - Import from `@modelcontextprotocol/ext-apps/server`
9. **useHostStyles(app)** - Required for theme integration in React

## Testing

1. Build the UI: `npm run build`
2. Start the server: `npm run serve`
3. Use cloudflared to expose: `npx cloudflared tunnel --url http://127.0.0.1:3001`
4. Connect to Claude/ChatGPT via the tunnel URL
5. Invoke the tool and verify UI renders

## Next Steps

After scaffolding:
- Add more tools: Design additional tools following the same pattern
- `/mcp-builder:validate` - Check for best practices
- `/mcp-builder:deploy` - Deploy to cloud platform
