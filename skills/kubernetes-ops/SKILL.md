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
mcporter call kubernetes.<tool_name> [json_arguments]
```

Example:
```bash
mcporter call kubernetes.events_list
mcporter call kubernetes.pods_list '{"namespace":"default"}'
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
mcporter call kubernetes.pods_list

# List pods in a specific namespace
mcporter call kubernetes.pods_list_in_namespace '{"namespace":"default"}'

# Get a specific pod
mcporter call kubernetes.pods_get '{"name":"my-pod","namespace":"default"}'

# Get pod logs (last 100 lines)
mcporter call kubernetes.pods_log '{"name":"my-pod","namespace":"default","tail":100}'

# Get previous container logs (for crashed pods)
mcporter call kubernetes.pods_log '{"name":"my-pod","namespace":"default","previous":true}'

# Resource usage for pods
mcporter call kubernetes.pods_top '{"namespace":"default"}'
```

### Execute commands in pods

```bash
mcporter call kubernetes.pods_exec '{"name":"my-pod","namespace":"default","command":["ls","-la","/tmp"]}'
```

### Generic Kubernetes resources (Deployments, Services, Ingress, etc.)

```bash
# List deployments
mcporter call kubernetes.resources_list '{"apiVersion":"apps/v1","kind":"Deployment","namespace":"default"}'

# Get a specific deployment
mcporter call kubernetes.resources_get '{"apiVersion":"apps/v1","kind":"Deployment","name":"my-app","namespace":"default"}'

# List services
mcporter call kubernetes.resources_list '{"apiVersion":"v1","kind":"Service","namespace":"default"}'

# List ingresses
mcporter call kubernetes.resources_list '{"apiVersion":"networking.k8s.io/v1","kind":"Ingress"}'

# List nodes
mcporter call kubernetes.resources_list '{"apiVersion":"v1","kind":"Node"}'

# List all resources with label selector
mcporter call kubernetes.resources_list '{"apiVersion":"apps/v1","kind":"Deployment","labelSelector":"app=nginx"}'
```

### Namespaces and events

```bash
# List all namespaces
mcporter call kubernetes.namespaces_list

# List events (all namespaces)
mcporter call kubernetes.events_list

# List events in a specific namespace
mcporter call kubernetes.events_list '{"namespace":"production"}'
```

### Node metrics

```bash
# Resource usage for all nodes
mcporter call kubernetes.nodes_top

# Resource usage for a specific node
mcporter call kubernetes.nodes_top '{"name":"worker-1"}'

# Detailed node stats
mcporter call kubernetes.nodes_stats_summary '{"name":"worker-1"}'
```

### Cluster configuration

```bash
# View current kubeconfig
mcporter call kubernetes.configuration_view '{"minified":true}'

# List available contexts
mcporter call kubernetes.configuration_contexts_list
```

### Scaling resources

```bash
# Get current scale
mcporter call kubernetes.resources_scale '{"apiVersion":"apps/v1","kind":"Deployment","name":"my-app","namespace":"default"}'

# Scale a deployment (NOTE: requires MCP server NOT in read-only mode)
mcporter call kubernetes.resources_scale '{"apiVersion":"apps/v1","kind":"Deployment","name":"my-app","namespace":"default","scale":3}'
```

## Discovering available tools

If unsure what tools are available, list them:

```bash
mcporter tools list
```

## Important notes

- The Kubernetes MCP Server runs in **read-only mode** by default. Write operations (create, update, delete, scale) will be rejected unless read-only mode is disabled.
- Always specify `namespace` when working with namespaced resources to avoid ambiguity.
- For `resources_list` and `resources_get`, you need both `apiVersion` and `kind`. Common combinations: `v1 Pod`, `v1 Service`, `v1 Node`, `apps/v1 Deployment`, `apps/v1 StatefulSet`, `networking.k8s.io/v1 Ingress`.
- Pod logs default to the last 100 lines. Use `tail` parameter to adjust.
- When diagnosing issues, check events first (`events_list`), then pod status, then logs.

## Alert auto-debug workflow

When you receive a message in a Telegram group that matches the alert pattern below, you MUST automatically start the debug workflow. Do NOT wait for additional instructions.

### How to recognize an alert

Alert messages from the alert-receiver service follow this pattern:
```
🚨 K8S ALERT: <alertname>
📌 Status: `FIRING`
⏰ Thời gian: ...
🔴 Severity: `critical`
📦 Namespace: `<namespace>`
🎯 Pod: `<pod>`
...
@openclaw_k8s_bot hãy debug alert này
```

When you see a message containing `🚨 K8S ALERT:` — treat it as an alert and begin debugging immediately.

### Debug procedure

Follow these steps **in order**. Execute ALL steps before writing your response:

**Step 1: Parse alert info**
Extract from the alert message:
- `alertname` — the alert name
- `namespace` — the K8s namespace
- `pod` — the pod name (may be `N/A` for non-pod alerts)
- `severity` — critical/warning/info
- `description` — what happened

**Step 2: Get pod details (if pod is specified)**
```bash
mcporter call kubernetes.pods_get '{"name":"<pod>","namespace":"<namespace>"}'
```
Look for: pod status, restart count, container statuses, conditions, reason for failure.

**Step 3: Get pod logs**
```bash
# Current logs (last 200 lines)
mcporter call kubernetes.pods_log '{"name":"<pod>","namespace":"<namespace>","tail":200}'
```
Look for: error messages, stack traces, connection failures, OOM messages.

**Step 4: Get previous container logs (if pod is crashing/restarting)**
```bash
mcporter call kubernetes.pods_log '{"name":"<pod>","namespace":"<namespace>","previous":true,"tail":200}'
```
This is critical for CrashLoopBackOff — the current container may not have useful logs.

**Step 5: Check events in the namespace**
```bash
mcporter call kubernetes.events_list '{"namespace":"<namespace>"}'
```
Look for: Warning events, FailedScheduling, FailedMount, BackOff, Unhealthy, OOMKilled.

**Step 6: Check the owning resource (Deployment/StatefulSet/DaemonSet)**
If the pod name follows a pattern like `<deployment>-<replicaset>-<hash>`:
```bash
# Extract deployment name from pod name (remove last 2 segments)
mcporter call kubernetes.resources_get '{"apiVersion":"apps/v1","kind":"Deployment","name":"<deployment>","namespace":"<namespace>"}'
```
Look for: replicas, update strategy, resource limits, image version.

**Step 7: Check resource usage (if applicable)**
```bash
mcporter call kubernetes.pods_top '{"namespace":"<namespace>"}'
```
Look for: CPU/memory usage close to limits (potential OOM).

### Alert-specific debug focus

Depending on the `alertname`, focus your investigation:

| Alert | Primary focus |
|-------|--------------|
| `PodCrashLooping` / `CrashLoopBackOff` | Steps 3-4 (logs + previous logs). Look for startup errors. |
| `PodNotReady` | Step 2 (pod conditions). Check readiness probe failures. |
| `OOMKilled` | Step 7 (resource usage) + Step 2 (container lastState). |
| `HighMemoryUsage` / `HighCPUUsage` | Step 7 (pods_top). Compare with resource limits. |
| `PodEvicted` | Step 5 (events). Check node pressure conditions. |
| `ImagePullBackOff` | Step 5 (events). Check image name, registry auth. |
| `FailedScheduling` | Step 5 (events). Check node resources, taints, affinity. |
| `VolumeMount` errors | Step 5 (events). Check PVC status, storage class. |

For alerts not in this table, run ALL steps and analyze based on collected data.

### Response format

After completing debug steps, respond in this format:

```
🔍 **DEBUG REPORT: <alertname>**

📋 **Thông tin alert:**
- Alert: `<alertname>`
- Namespace: `<namespace>`
- Pod: `<pod>`
- Severity: `<severity>`

🔎 **Phân tích nguyên nhân:**
<Describe root cause based on collected evidence from logs, events, pod status>

📊 **Bằng chứng:**
- Pod status: <status>
- Restart count: <count>
- Key log lines: <relevant log excerpts>
- Related events: <relevant events>

🛠️ **Cách fix đề xuất:**
1. <First recommended fix>
2. <Second recommended fix if applicable>
3. <Additional steps if needed>

⚠️ **Lưu ý:** Bot chỉ đề xuất, KHÔNG tự động thực hiện thay đổi trên hệ thống. Vui lòng review và thực hiện thủ công.
```

### Important rules for alert debugging

1. **NEVER modify the cluster** — read-only mode. Only suggest fixes.
2. **Always respond in Vietnamese** — this is the user preference.
3. **Be specific** — cite actual log lines, event messages, and status values.
4. **Be actionable** — suggest concrete kubectl commands the user can run to fix.
5. **Handle missing data gracefully** — if a pod doesn't exist or logs are empty, say so and adjust your analysis.
6. **For RESOLVED alerts** (status = `RESOLVED`) — briefly acknowledge the resolution, no need for full debug.
7. **Group multiple alerts** — if multiple alerts arrive close together for the same pod/deployment, do ONE consolidated debug report.
