# MCP Gateway — UWCU

UWCU fork of [microsoft/mcp-gateway](https://github.com/microsoft/mcp-gateway). A reverse proxy and management layer for [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) servers on NKP.

**Upstream README:** [UPSTREAM-README.md](UPSTREAM-README.md)

## What It Does

The MCP Gateway provides a single entry point for all MCP servers at UWCU. Instead of clients connecting to each MCP server individually, they connect to the gateway, which handles:

- **Routing** — direct access to any registered server via `/adapters/{name}/mcp`, or smart routing via `/mcp` (tool router knows all registered tools)
- **Session affinity** — stateful MCP conversations stick to the same server instance
- **Lifecycle management** — register, update, and decommission MCP servers via REST API
- **Auth (future)** — Entra ID bearer token auth with role-based access control (`mcp.admin`, `mcp.engineer`)

## Architecture

```
Clients (VS Code, Claude Code, uChat)
  |
  v
mcp-gateway.nkp-np.uwcu.org  (Traefik Ingress)
  |
  v
mcpgateway pod  (auth, session routing, control plane API)
  |                                     |
  |  /adapters/{name}/mcp              |  /mcp
  v                                     v
mcp-proxy pod  (per-server)            toolgateway pod  (smart router)
  |                                     |
  v                                     v
actual MCP server                      routes to registered tools
(in mcp-servers-np)
```

**How the proxy works:** The gateway manages pods in its own namespace (`mcp-gateway-np`). Our actual MCP servers live in `mcp-servers-np` with their secrets and config. The `mcp-proxy` image is a lightweight Python bridge — the gateway creates a proxy pod per server, and each proxy forwards traffic to the real server via `MCP_PROXY_URL`.

## Components

| Pod | Image | Role |
|-----|-------|------|
| mcpgateway | `mcp-gateway/mcpgateway-service` | Main gateway — API, routing, auth |
| toolgateway | `mcp-gateway/toolgateway` | Smart tool router — single `/mcp` endpoint for all tools |
| redis | `redis:7-alpine` | Adapter/tool metadata store + session cache |
| {name}-0 | `mcp-gateway/mcp-proxy` | Per-server proxy (one per registered adapter) |

## Registered MCP Servers

| Name | Description | Backend URL |
|------|-------------|-------------|
| nutanix-mcp | Nutanix Prism Central | `http://nutanix-mcp.mcp-servers-np:8080/mcp` |
| openrouter-image-mcp | OpenRouter image generation | `http://openrouter-image-mcp.mcp-servers-np:3001/mcp` |
| exa-search-mcp | Exa web search | `http://exa-search-mcp.mcp-servers-np:8000/mcp` |
| ivanti-mcp | Ivanti ITSM | `http://ivanti-mcp.mcp-servers-np:8090/mcp` |
| ado-mcp | Azure DevOps | `http://ado-mcp.mcp-servers-np:8055/mcp` |
| logicmonitor-mcp | LogicMonitor monitoring | `http://logicmonitor-mcp.mcp-servers-np:3000/mcp` |

## How to Use

### Connect from VS Code

Add to `.vscode/mcp.json`:

```json
{
  "servers": {
    "nutanix": {
      "url": "https://mcp-gateway.nkp-np.uwcu.org/adapters/nutanix-mcp/mcp"
    },
    "all-tools": {
      "url": "https://mcp-gateway.nkp-np.uwcu.org/mcp"
    }
  }
}
```

- `/adapters/{name}/mcp` — connect to a specific server
- `/mcp` — connect to the tool router (knows all registered tools, routes automatically)

### Connect from Claude Code

```json
{
  "mcpServers": {
    "nutanix": {
      "url": "https://mcp-gateway.nkp-np.uwcu.org/adapters/nutanix-mcp/mcp"
    }
  }
}
```

### Test with curl

```bash
curl -sk -X POST https://mcp-gateway.nkp-np.uwcu.org/adapters/nutanix-mcp/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

## Managing Adapters

### List all registered servers

```bash
curl -sk https://mcp-gateway.nkp-np.uwcu.org/adapters
```

### Register a new server

```bash
curl -sk -X POST https://mcp-gateway.nkp-np.uwcu.org/adapters \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-new-mcp",
    "imageName": "mcp-gateway/mcp-proxy",
    "imageVersion": "latest",
    "environmentVariables": {
      "MCP_PROXY_URL": "http://my-service.my-namespace.svc.cluster.local:8080/mcp"
    },
    "description": "Description of the MCP server"
  }'
```

### Check adapter status

```bash
curl -sk https://mcp-gateway.nkp-np.uwcu.org/adapters/nutanix-mcp/status
```

### View adapter logs

```bash
curl -sk https://mcp-gateway.nkp-np.uwcu.org/adapters/nutanix-mcp/logs
```

### Remove an adapter

```bash
curl -sk -X DELETE https://mcp-gateway.nkp-np.uwcu.org/adapters/nutanix-mcp
```

### Full API reference

Import `openapi/mcp-gateway.openapi.json` into Postman or Bruno.

## Fork Changes from Upstream

We maintain a minimal fork of [microsoft/mcp-gateway](https://github.com/microsoft/mcp-gateway) with four changes:

### 1. Redis replaces Cosmos DB

**Files:** `dotnet/Microsoft.McpGateway.Service/src/Program.cs`, `dotnet/Microsoft.McpGateway.Tools/src/Program.cs`

Upstream requires Azure Cosmos DB in production mode. We use Redis in all environments. Entra ID auth is preserved in production mode — only the storage backend changed.

**Why:** On-prem NKP, no Azure Cosmos DB. Redis is lightweight and runs alongside the gateway.

### 2. Configurable K8s namespace

**Files:** `dotnet/Microsoft.McpGateway.Service/src/Program.cs`, `dotnet/Microsoft.McpGateway.Service/src/Routing/AdapterKubernetesNodeInfoProvider.cs`, `dotnet/Microsoft.McpGateway.Management/src/Deployment/KubernetesAdapterDeploymentManager.cs`

Upstream hardcodes `"adapter"` as the namespace for managed pods. We made it configurable via `AdapterNamespace` environment variable (defaults to `"adapter"`).

**Why:** NKP uses `mcp-gateway-np` as the project namespace.

### 3. Dockerfiles + ADO pipeline

**Files:** `dotnet/Microsoft.McpGateway.Service/Dockerfile`, `dotnet/Microsoft.McpGateway.Tools/Dockerfile`, `sample-servers/mcp-proxy/Dockerfile`, `azure-pipelines.yml`

Upstream uses `dotnet publish` with Visual Studio publish profiles. We added multi-stage Dockerfiles and an Azure Pipelines definition with change detection (only rebuilds affected images).

### 4. Kustomize manifests

**Directory:** `base/`, `overlays/`

K8s deployment manifests for NKP. FluxCD syncs from `overlays/dev/` to `mcp-gateway-np` on the nonprod cluster.

## Syncing with Upstream

```bash
# From the local clone at /opt/nkp/nkp-v2.17.1/mcp-gateway-upstream/
git fetch upstream
git merge upstream/main
# Resolve conflicts — typically in Program.cs and KubernetesAdapterDeploymentManager.cs
git push origin main   # GitHub fork
git push ado main      # ADO (triggers pipeline)
```

Remotes:
- `upstream` — `https://github.com/microsoft/mcp-gateway.git` (Microsoft)
- `origin` — `https://github.com/mathomas95/mcp-gateway.git` (UWCU GitHub fork)
- `ado` — `https://dev.azure.com/UWCU/Infrastructure%20Services/_git/mcp-gateway` (builds)

## Enabling Entra ID Auth

Currently running in Development mode (no auth). To switch to Production with Entra ID:

1. Create an Entra ID app registration (see [docs/entra-app-roles.md](docs/entra-app-roles.md))
2. Update `base/config/gateway-config.yaml`:
   ```yaml
   ASPNETCORE_ENVIRONMENT: "Production"
   AzureAd__Instance: "https://login.microsoftonline.com/"
   AzureAd__TenantId: "<tenant-id>"
   AzureAd__ClientId: "<client-id>"
   AzureAd__Audience: "<client-id>"
   ```
3. Push to ADO, FluxCD deploys, gateway restarts with auth enabled
4. Clients acquire bearer tokens: `az account get-access-token --resource <client-id>`

## Environment

| Item | Value |
|------|-------|
| Cluster | fit-nkp-nonprod |
| Gateway namespace | mcp-gateway-np |
| MCP servers namespace | mcp-servers-np |
| Ingress (nonprod) | mcp-gateway.nkp-np.uwcu.org |
| Ingress (prod) | mcp-gateway.uwcu.org (Phase 9) |
| Harbor project | nkp-mgmt.uwcu.org:5000/mcp-gateway/* |
| Pipeline | NKP-Agents pool, change detection |
| Auth | Development (no auth) — Entra ID ready |
| Storage | Redis (in-namespace) |
