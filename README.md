# OpenClaw K8s Operations Bot

Automate Kubernetes cluster operations through Telegram using OpenClaw as the AI gateway and ChatGPT as the reasoning engine.

## Architecture

```
User (Telegram) --> OpenClaw Gateway --> OpenAI ChatGPT API
                         |
                         v (MCP Protocol / SSE)
                  Kubernetes MCP Server --> K8s Cluster
```

**Components:**

| Service | Role | Image |
|---------|------|-------|
| OpenClaw | AI gateway, Telegram integration, tool orchestration | `node:20-alpine` + openclaw npm |
| Kubernetes MCP Server | K8s API bridge via Model Context Protocol | `ghcr.io/containers/kubernetes-mcp-server:latest` |
| OpenAI ChatGPT | AI reasoning and tool selection | External API |

## Prerequisites

- Docker and Docker Compose v2+
- An OpenAI API key (from [platform.openai.com](https://platform.openai.com/api-keys))
- A Telegram bot token (from [@BotFather](https://t.me/BotFather))
- Your Telegram user ID (from [@userinfobot](https://t.me/userinfobot))
- A valid `kubeconfig` file with access to your Kubernetes cluster

## Quick Start

### 1. Clone and configure

```bash
git clone <repo-url> openclaw-bot
cd openclaw-bot

# Create your environment file from the template
cp .env.example .env
```

Edit `.env` with your actual values:

```bash
OPENAI_API_KEY=sk-your-actual-key
TELEGRAM_BOT_TOKEN=your-bot-token-from-botfather
TELEGRAM_ALLOWED_USERS=your-telegram-user-id
```

### 2. Add your kubeconfig

Copy your Kubernetes config into the project:

```bash
# Option A: Copy from default location
cp ~/.kube/config kubeconfig/config

# Option B: Export from a specific context
kubectl config view --minify --flatten > kubeconfig/config
```

### 3. Configure Telegram bot settings

Edit `config/openclaw.json` and replace the placeholder values:

```jsonc
{
  "channels": {
    "telegram": {
      "botToken": "YOUR_ACTUAL_BOT_TOKEN",
      "allowFrom": ["YOUR_TELEGRAM_USER_ID"]
    }
  }
}
```

> Alternatively, if OpenClaw supports environment variable interpolation (`${VAR}`), the defaults from `.env` will be used.

### 4. Start the services

```bash
docker compose up -d
```

### 5. Verify

```bash
# Check both services are running
docker compose ps

# Check OpenClaw logs
docker compose logs -f openclaw

# Check K8s MCP Server logs
docker compose logs -f kubernetes-mcp-server
```

### 6. Chat with your bot

Open Telegram and message your bot. Try commands like:

- "List all namespaces in the cluster"
- "Show pods in the production namespace"
- "Get events with warnings"
- "Show resource usage for all nodes"
- "Describe the deployment named nginx in default namespace"
- "Show logs for pod my-app-xyz in staging namespace"

## Configuration Reference

### Environment Variables (`.env`)

| Variable | Description | Required |
|----------|-------------|----------|
| `OPENAI_API_KEY` | OpenAI API key for ChatGPT | Yes |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token from @BotFather | Yes |
| `TELEGRAM_ALLOWED_USERS` | Comma-separated Telegram user IDs | Yes |

### OpenClaw Config (`config/openclaw.json`)

Key sections:

- **identity** - Bot name and personality theme
- **agents.defaults.model** - AI model selection (default: `openai/gpt-4o`)
- **channels.telegram** - Telegram bot settings and access control
- **mcp.servers.kubernetes** - Kubernetes MCP Server connection URL
- **gateway** - Gateway port and UI settings

### Kubernetes MCP Server

The MCP server runs in **read-only mode** by default for safety. To enable write operations (create, update, delete resources), remove `--read-only` from the command in `docker-compose.yml`:

```yaml
kubernetes-mcp-server:
  command:
    - "--port"
    - "8089"
    # - "--read-only"    # Commented out to enable write operations
    - "--log-level"
    - "2"
```

To enable Helm operations, add the `--toolsets` flag:

```yaml
kubernetes-mcp-server:
  command:
    - "--port"
    - "8089"
    - "--read-only"
    - "--toolsets"
    - "core,config,helm"
    - "--log-level"
    - "2"
```

## Available K8s Operations

With `containers/kubernetes-mcp-server`, these tools are available to the AI:

| Category | Tools | Description |
|----------|-------|-------------|
| Pods | `pods_list`, `pods_get`, `pods_log`, `pods_exec`, `pods_top` | List, inspect, get logs, exec, and check resource usage |
| Resources | `resources_list`, `resources_get`, `resources_create_or_update`, `resources_delete` | CRUD on any K8s resource |
| Events | `events_list` | View cluster events for debugging |
| Namespaces | `namespaces_list` | List all namespaces |
| Nodes | `nodes_top`, `nodes_log` | Node resource metrics and logs |
| Config | `configuration_view`, `configuration_contexts_list` | View kubeconfig and contexts |
| Helm | `helm_install`, `helm_list`, `helm_uninstall` | Manage Helm releases (requires `helm` toolset) |

## Operations

### View logs

```bash
docker compose logs -f                    # All services
docker compose logs -f openclaw           # OpenClaw only
docker compose logs -f kubernetes-mcp-server  # K8s MCP only
```

### Restart services

```bash
docker compose restart                    # All
docker compose restart openclaw           # OpenClaw only
```

### Stop

```bash
docker compose down                       # Stop and remove containers
docker compose down -v                    # Also remove volumes (loses state)
```

### Update images

```bash
docker compose pull
docker compose up -d
```

## Security Considerations

- **kubeconfig**: Mounted read-only into the MCP server container. Never committed to git.
- **API keys**: Stored in `.env` which is git-ignored. Never hardcode in config files.
- **Read-only mode**: K8s MCP server defaults to read-only, preventing accidental cluster changes.
- **Network isolation**: K8s MCP server is only accessible via Docker internal network (`expose`, not `ports`).
- **Telegram access control**: Only whitelisted user IDs can interact with the bot.
- **No new privileges**: Both containers run with `no-new-privileges` security option.
- **Resource limits**: CPU and memory limits are set to prevent resource exhaustion.

## Troubleshooting

### OpenClaw not starting

```bash
docker compose logs openclaw
```

Common issues:
- Missing or invalid `OPENAI_API_KEY`
- Invalid `openclaw.json` syntax (use a JSON5 validator)
- Port 18789 already in use

### K8s MCP Server connection issues

```bash
docker compose logs kubernetes-mcp-server
```

Common issues:
- Invalid or missing kubeconfig at `kubeconfig/config`
- Kubeconfig references `localhost` or `127.0.0.1` which won't work from inside Docker (use the actual cluster API server address)
- Cluster API server not reachable from Docker network

### Bot not responding on Telegram

- Verify the bot token is correct
- Check that your Telegram user ID is in the `allowFrom` list
- Ensure OpenClaw gateway is healthy: `docker compose ps`

## References

### OpenClaw - Personal AI Assistant

- Website: [https://openclaw.ai](https://openclaw.ai/)
- GitHub: [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- Documentation: [https://docs.openclaw.ai](https://docs.openclaw.ai)

Open-source personal AI assistant that runs on your machine. Supports WhatsApp, Telegram, Discord, Slack, Signal, iMessage. Works with Anthropic, OpenAI, Google, local models. Extensible via skills and MCP servers.

### Kubernetes MCP Server (containers/kubernetes-mcp-server)

- GitHub: [https://github.com/containers/kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)
- Docker image: `ghcr.io/containers/kubernetes-mcp-server:latest`
- License: Apache-2.0

Go-based native MCP server for Kubernetes and OpenShift. Interacts directly with the K8s API server without requiring kubectl or any external dependencies. Supports multi-cluster, read-only mode, Helm, and SSE/HTTP transport. **This is the MCP server used in this project.**

### Kubernetes MCP Server (Flux159/mcp-server-kubernetes)

- GitHub: [https://github.com/Flux159/mcp-server-kubernetes](https://github.com/Flux159/mcp-server-kubernetes)
- npm: [mcp-server-kubernetes](https://www.npmjs.com/package/mcp-server-kubernetes)
- License: MIT

Node.js/TypeScript-based MCP server wrapping kubectl and helm CLI. Provides unified kubectl API, Helm operations, pod cleanup, node management, and troubleshooting prompts. Alternative option if you prefer a kubectl-based approach.

### Model Context Protocol (MCP)

- Specification: [https://modelcontextprotocol.io](https://modelcontextprotocol.io)
