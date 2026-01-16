---
name: mcp-builder:deploy
description: Generate deployment configuration for Cloud Run, Render, or Fly.io
---

# Deploy MCP Server

You are helping the user deploy their MCP server to production.

## Deployment Options

| Platform | Best For | Complexity | Cost |
|----------|----------|------------|------|
| **Google Cloud Run** | Auto-scaling, enterprise | Medium | Pay-per-use |
| **Render** | Simple deploys, small teams | Low | $7+/month |
| **Fly.io** | Edge deployment, low latency | Medium | Pay-per-use |

## Workflow

### Phase 1: Choose Platform

Ask the user:
1. What's your expected traffic volume?
2. Do you need edge deployment (low latency globally)?
3. What's your budget?
4. Do you already use a cloud provider?

### Phase 2: Prepare for Deployment

**Verify server is production-ready:**
- [ ] Environment variables externalized
- [ ] No hardcoded secrets
- [ ] Health check endpoint exists
- [ ] Logging configured
- [ ] Error handling complete

---

## Google Cloud Run

### Dockerfile

```dockerfile
# Python FastMCP Server
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Cloud Run sets PORT environment variable
ENV PORT=8080
EXPOSE 8080

# Start server
CMD ["python", "server.py"]
```

### requirements.txt

```
fastmcp>=0.1.0
uvicorn>=0.30.0
python-dotenv>=1.0.0
httpx>=0.27.0
# Add your other dependencies
```

### Deploy Script

```bash
#!/bin/bash
# deploy-cloud-run.sh

PROJECT_ID="your-project-id"
REGION="us-central1"
SERVICE_NAME="your-mcp-server"

# Build and push image
gcloud builds submit --tag gcr.io/$PROJECT_ID/$SERVICE_NAME

# Deploy to Cloud Run
gcloud run deploy $SERVICE_NAME \
  --image gcr.io/$PROJECT_ID/$SERVICE_NAME \
  --platform managed \
  --region $REGION \
  --allow-unauthenticated \
  --set-env-vars "NODE_ENV=production" \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \
  --max-instances 10

# Get the URL
gcloud run services describe $SERVICE_NAME \
  --region $REGION \
  --format 'value(status.url)'
```

### Cloud Run Tips

- Set `--min-instances 1` to avoid cold starts
- Use `--cpu-boost` for faster cold starts
- Set `--concurrency` based on your server's capacity
- Disable HTTP/2 if you have issues: `--use-http2=false`

---

## Render

### render.yaml

```yaml
services:
  - type: web
    name: your-mcp-server
    runtime: python
    buildCommand: pip install -r requirements.txt
    startCommand: python server.py
    envVars:
      - key: PORT
        value: 10000
      - key: NODE_ENV
        value: production
      - key: PYTHON_VERSION
        value: 3.11.0
    healthCheckPath: /health
    autoDeploy: true
```

### Deploy Steps

1. Push code to GitHub
2. Connect repo to Render
3. Render auto-detects `render.yaml`
4. Set environment variables in dashboard
5. Deploy

### Render Tips

- Use "Web Service" type, not "Background Worker"
- Health check keeps service awake
- Auto-deploy on push to main branch

---

## Fly.io

### fly.toml

```toml
app = "your-mcp-server"
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 0

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory_mb = 512
```

### Dockerfile (same as Cloud Run)

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV PORT=8080
EXPOSE 8080
CMD ["python", "server.py"]
```

### Deploy Script

```bash
#!/bin/bash
# deploy-fly.sh

# Install Fly CLI if needed
# curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# Create app (first time only)
fly apps create your-mcp-server

# Set secrets
fly secrets set AUTH0_DOMAIN=your-domain.auth0.com
fly secrets set AUTH0_AUDIENCE=https://your-api.com

# Deploy
fly deploy

# Check status
fly status
```

### Fly.io Tips

- Use `--ha` for high availability (2+ machines)
- Deploy to multiple regions for global low latency
- `fly scale count 2` for redundancy

---

## Environment Variables

**Set these in your deployment:**

```bash
# Required
PORT=8080
NODE_ENV=production

# If using Auth
AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_AUDIENCE=https://your-api.com

# If using database
DATABASE_URL=postgresql://user:pass@host:5432/db

# Your API keys (keep secret!)
OPENAI_API_KEY=sk-...
```

---

## Post-Deployment Checklist

- [ ] Health endpoint responds: `curl https://your-server.com/health`
- [ ] Tools list correctly: `curl -X POST https://your-server.com/mcp ...`
- [ ] Test one tool call end-to-end
- [ ] Configure custom domain (optional)
- [ ] Set up monitoring/alerts
- [ ] Test with real MCP client (Claude Desktop, Cursor)

## Update State

After deployment, update `.mcp-builder/state.json`:

```json
{
  "deployment": {
    "platform": "cloud-run",
    "url": "https://your-server-xyz.run.app",
    "region": "us-central1",
    "deployedAt": "2025-01-15T12:00:00Z"
  }
}
```
