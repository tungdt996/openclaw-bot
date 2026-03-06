---
name: kubernetes-ops
description: Query and manage Kubernetes clusters via mcporter connected to a Kubernetes MCP Server
metadata: {"openclaw":{"requires":{"bins":["mcporter"]},"emoji":"⎈"}}
---

# Kubernetes Operations via mcporter

You have access to a Kubernetes cluster through `mcporter`, which connects to a Kubernetes MCP Server.
Use the `exec` tool to run mcporter commands. The MCP server named **kubernetes** is pre-configured.

## How to call Kubernetes tools

Use this pattern to call any Kubernetes MCP tool:

```bash
mcporter tool call kubernetes <tool_name> '<json_arguments>'
```

## Output formatting rules

The MCP server returns raw ASCII tables. NEVER send raw table output directly to the user.
Always parse the output and reformat it into clean, readable Markdown suitable for Telegram.

### Formatting guidelines

- Use **bold** for resource names and key identifiers
- Use `monospace` for namespaces, status values, versions, and IPs
- Use bullet lists instead of tables
- Use status emoji:
  - ✅ Running, Active, Healthy, Ready, Succeeded
  - ❌ Failed, Error, CrashLoopBackOff, OOMKilled
  - ⏳ Pending, ContainerCreating, Terminating, Initializing
  - ⚠️ Warning, Unknown, NotReady, Evicted
- When listing >10 items, group by namespace or status and show counts
- For errors and warnings, explain the likely cause and suggest a fix
- Respond in Vietnamese (user preference)

### Formatting examples

Raw table from MCP:
```
APIVERSION   KIND   NAME              STATUS   AGE
v1           Pod    nginx-abc123      Running  2d
v1           Pod    redis-xyz789      Failed   1h
v1           Pod    app-server-def    Pending  5m
```

Formatted response:
```
⎈ **Pods trong namespace `default`** (3 pods)

✅ **nginx-abc123** — `Running` (2 ngay)
❌ **redis-xyz789** — `Failed` (1 gio)
⏳ **app-server-def** — `Pending` (5 phut)

📊 Tong ket: 1 Running, 1 Failed, 1 Pending
```

Raw event list from MCP:
```
LAST SEEN   TYPE      REASON    OBJECT          MESSAGE
2m          Warning   BackOff   pod/redis-xyz   Back-off restarting failed container
5m          Normal    Pulled    pod/nginx-abc   Successfully pulled image "nginx:latest"
```

Formatted response:
```
⚠️ **Events gan day**

🔴 **Warning** — pod `redis-xyz` (2 phut truoc)
   Back-off restarting failed container
   → Kiem tra logs: co the container bi crash do loi config hoac OOM

🟢 **Normal** — pod `nginx-abc` (5 phut truoc)
   Pull image `nginx:latest` thanh cong
```

### Handling broad questions

When the user asks general questions like "cluster co on khong?" or "check cluster health":
1. Run `namespaces_list` to verify API connectivity
2. Run `pods_list` to check overall pod status
3. Run `events_list` to find recent warnings/errors
4. Run `nodes_top` to check resource usage
5. Summarize everything in one consolidated response with health status

## Available tools

### List and inspect pods

```bash
# List all pods across all namespaces
mcporter tool call kubernetes pods_list '{}'

# List pods in a specific namespace
mcporter tool call kubernetes pods_list_in_namespace '{"namespace":"default"}'

# Get a specific pod
mcporter tool call kubernetes pods_get '{"name":"my-pod","namespace":"default"}'

# Get pod logs (last 100 lines)
mcporter tool call kubernetes pods_log '{"name":"my-pod","namespace":"default","tail":100}'

# Get previous container logs (for crashed pods)
mcporter tool call kubernetes pods_log '{"name":"my-pod","namespace":"default","previous":true}'

# Resource usage for pods
mcporter tool call kubernetes pods_top '{"namespace":"default"}'
```

### Execute commands in pods

```bash
mcporter tool call kubernetes pods_exec '{"name":"my-pod","namespace":"default","command":["ls","-la","/tmp"]}'
```

### Generic Kubernetes resources (Deployments, Services, Ingress, etc.)

```bash
# List deployments
mcporter tool call kubernetes resources_list '{"apiVersion":"apps/v1","kind":"Deployment","namespace":"default"}'

# Get a specific deployment
mcporter tool call kubernetes resources_get '{"apiVersion":"apps/v1","kind":"Deployment","name":"my-app","namespace":"default"}'

# List services
mcporter tool call kubernetes resources_list '{"apiVersion":"v1","kind":"Service","namespace":"default"}'

# List ingresses
mcporter tool call kubernetes resources_list '{"apiVersion":"networking.k8s.io/v1","kind":"Ingress"}'

# List nodes
mcporter tool call kubernetes resources_list '{"apiVersion":"v1","kind":"Node"}'

# List all resources with label selector
mcporter tool call kubernetes resources_list '{"apiVersion":"apps/v1","kind":"Deployment","labelSelector":"app=nginx"}'
```

### Namespaces and events

```bash
# List all namespaces
mcporter tool call kubernetes namespaces_list '{}'

# List events (all namespaces)
mcporter tool call kubernetes events_list '{}'

# List events in a specific namespace
mcporter tool call kubernetes events_list '{"namespace":"production"}'
```

### Node metrics

```bash
# Resource usage for all nodes
mcporter tool call kubernetes nodes_top '{}'

# Resource usage for a specific node
mcporter tool call kubernetes nodes_top '{"name":"worker-1"}'

# Detailed node stats
mcporter tool call kubernetes nodes_stats_summary '{"name":"worker-1"}'
```

### Cluster configuration

```bash
# View current kubeconfig
mcporter tool call kubernetes configuration_view '{"minified":true}'

# List available contexts
mcporter tool call kubernetes configuration_contexts_list '{}'
```

### Scaling resources

```bash
# Get current scale
mcporter tool call kubernetes resources_scale '{"apiVersion":"apps/v1","kind":"Deployment","name":"my-app","namespace":"default"}'

# Scale a deployment (NOTE: requires MCP server NOT in read-only mode)
mcporter tool call kubernetes resources_scale '{"apiVersion":"apps/v1","kind":"Deployment","name":"my-app","namespace":"default","scale":3}'
```

## Discovering available tools

If unsure what tools are available, list them:

```bash
mcporter tools list kubernetes
```

## Important notes

- The Kubernetes MCP Server runs in **read-only mode** by default. Write operations (create, update, delete, scale) will be rejected unless read-only mode is disabled.
- Always specify `namespace` when working with namespaced resources to avoid ambiguity.
- For `resources_list` and `resources_get`, you need both `apiVersion` and `kind`. Common combinations: `v1 Pod`, `v1 Service`, `v1 Node`, `apps/v1 Deployment`, `apps/v1 StatefulSet`, `networking.k8s.io/v1 Ingress`.
- Pod logs default to the last 100 lines. Use `tail` parameter to adjust.
- When diagnosing issues, check events first (`events_list`), then pod status, then logs.
